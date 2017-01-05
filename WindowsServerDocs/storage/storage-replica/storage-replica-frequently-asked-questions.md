---
title: Frequently Asked Questions about Storage Replica
ms.prod: windows-server-threshold
manager: siroy
ms.author: nedpyle
ms.technology: storage-replica
ms.topic: get-started-article
author: nedpyle
ms.date: 10/13/2016
ms.assetid: 12bc8e11-d63c-4aef-8129-f92324b2bf1b
---
# Frequently Asked Questions about Storage Replica

>Applies To: Windows Server 2016

This topic contains answers to frequently asked questions (FAQs) about Storage Replica.

## <a name="FAQ1"></a> Is Storage Replica supported on Nano Server?  
Yes.  

> [!NOTE]
> You must use the **Storage** Nano Server package during setup. For more information about deploying Nano Server, see [Getting Started with Nano Server](https://technet.microsoft.com/library/mt126167.aspx).  

Install Storage Replica on Nano Server using  PowerShell remoting as follows:  

1. Add the Nano server to your client trust list.  
   > [!NOTE]
   > This step is only necessary if the computer is not a member of an Active Directory Domain Services forest or in an untrusted forest. It adds NTLM support to PSSession remoting, which is disabled by default for security reasons. For more information, see [PowerShell Remoting Security Considerations](https://msdn.microsoft.com/powershell/scriptiwinrmsecurity).  

   ```  
       Set-Item WSMan:\localhost\Client\TrustedHosts "<computer name of Nano Server>"  
   ```  
2.  To install the Storage Replica feature, run the following cmdlet from a management computer:  

    ```  
    Install-windowsfeature -Name storage-replica,RSAT-Storage-Replica -ComputerName <nano server> -Restart -IncludeManagementTools  
    ```  

    Using the `Test-SRTopology` cmdlet with Nano Server in Windows Server 2016 requires remote script invocation with CredSSP. Unlike other Storage Replica cmdlets, `Test-SRTopology` requires running locally on the source server.   
On the Nano server (through a remote PSSession) :  

    >[!NOTE]
    >CREDSSP is needed for Kerberos double-hop support in the `Test-SRTopology` cmdlet, and not needed by other Storage Replica cmdlets, which handle distributed system credentials automatically. Using CREDSSP is not recommended under typical circumstances. For an alternative to CREDSSP, review the following Microsoft blog post: "PowerShell Remoting Kerberos Double Hop Solved Securely" - https://blogs.technet.microsoft.com/ashleymcglone/2016/08/30/powershell-remoting-kerberos-double-hop-solved-securely/ 

        Enable-WSManCredSSP -role server       

    On the management computer:  

         Enable-WSManCredSSP Client -DelegateComputer <remote server name>  

         $CustomCred = Get-Credential  

         Invoke-Command -ComputerName sr-srv01 -ScriptBlock { Test-SRTopology <commands> } -Authentication Credssp -Credential $CustomCred  

       Then copy the results to your management computer or share the path. Because Nano lacks the necessary graphical libraries, you can use Test-SRTopology to process the results and give you a report file with charts. For example:  

        Test-SRTopology -GenerateReport -DataPath \\sr-srv05\c$\temp  


## <a name="FAQ2"></a> How do I see the progress of replication during initial sync?  
The Event 1237 messages shown in the Storage Replica Admin even log on the destination server show number of bytes copied and bytes remaining every 10 seconds. You can also use the Storage Replica performance counter on the destination showing **\Storage Replica Statistics\Total Bytes Received** for one or more replicated volumes. You can also query the replication group using Windows PowerShell. For instance,  this sample command gets the name of the groups on the destination  then queries one group named **Replication 2** every 10 seconds to show progress:  

```  
Get-SRGroup

do{
    $r=(Get-SRGroup -Name "Replication 2").replicas
    [System.Console]::Write("Number of remaining bytes {0}`n", $r.NumOfBytesRemaining)
    Start-Sleep 10
}until($r.ReplicationStatus -eq 'ContinuouslyReplicating')
Write-Output "Replica Status: "$r.replicationstatus

```  

## <a name="FAQ3"></a>Can I specify specific network interfaces to be used for replication?  

Yes, using `Set-SRNetworkConstraint`. This cmdlet operates at the interface layer and be used on both cluster and non-cluster scenarios.  
For example, with a standalone server (on each node):  

```  
Get-SRPartnership  

Get-NetIPConfiguration  
```  
Note the gateway and interface information (on both servers) and the partnership directions. Then run:  

```  
Set-SRNetworkConstraint -SourceComputerName sr-srv06 -SourceRGName rg02 -  
SourceNWInterface 2 -DestinationComputerName sr-srv05 -DestinationNWInterface 3 -DestinationRGName rg01  

Get-SRNetworkConstraint  

Update-SmbMultichannelConnection  

```  

For configuring network constraints on a stretch cluster:  

    Set-SRNetworkConstraint -SourceComputerName sr-srv01 -SourceRGName group1 -SourceNWInterface "Cluster Network 1","Cluster Network 2" -DestinationComputerName sr-srv03 -DestinationRGName group2 -DestinationNWInterface "Cluster Network 1","Cluster Network 2"  

## <a name="FAQ4"></a> Can I configure one-to-many replication or transitive (A to B to C) replication?  
Not in Windows Server 2016. This release only supports one to one replication of a server, cluster, or stretch cluster node. This may change in a later release. You can of course configure replication between various servers of a specific volume pair, in either direction. For instance, Server 1 can replicate its D volume to server 2, and its E volume from Server 3.

## <a name="FAQ5"></a> Can I expand or shrink replicated volumes replicated by Storage Replica?  
You can expand (grow) volumes, but not shrink them. This is prevented by default; use the `Set-SRGroup -AllowVolumeResize $TRUE` option to allow it on both groups, prior to resizing.

Note that ReFS does not support shrinking drives regardless of replication, unlike NTFS.

## <a name="FAQ6"></a>Can I bring a destination volume online for read-only access?  
Not in Windows Server 2016. Storage Replica dismounts the destination volume and its drive letter or mount point when replication begins. This may change in a later release.  

## <a name="FAQ7"></a> Can I configure Scale-out File Server (SOFS) in a stretch cluster?  
While technically possible, this is not a recommended configuration in Windows Server 2016 due to the lack of site awareness in the compute nodes contacting the SOFS. If using campus-distance networking, where latencies are typically sub-millisecond, this configuration typically works works without issues.   

If configuring cluster-to-cluster replication, Storage Replica fully supports Scale-out File Servers, including the use of Storage Spaces Direct, when replicating between two clusters.  

## <a name="FAQ8"></a>Can I configure Storage Spaces Direct in a stretch cluster with Storage Replica?  
This is not a supported configuration in Windows Server 2016.  This may change in a later release. If configuring cluster-to-cluster replication, Storage Replica fully supports Scale Out File Servers and Hyper-V Servers, including the use of Storage Spaces Direct.  

## <a name="FAQ9"></a>How do I configure asynchronous replication?  

Specify `New-SRPartnership -ReplicationMode` and provide argument **Asynchronous**. By default, all replication in Storage Replica is synchronous. You can also change the mode with `Set-SRPartnership -ReplicationMode`.  

## <a name="FAQ10"></a>How do I prevent automatic failover of a stretch cluster?  
To prevent automatic failover, you can use PowerShell to configure `Get-ClusterNode -Name "NodeName").NodeWeight=0`. This removes the vote on each node in the disaster recovery site. Then you can use `Start-ClusterNode -PreventQuorum` on nodes in the primary site and `Start-ClusterNode -ForceQuorum` on nodes in the disaster site to force failover. There is no graphical option for preventing automatic failover, and preventing automatic failover is not recommended.  

## <a name="FAQ11"></a>How do I disable virtual machine resiliency?  
To prevent the new Hyper-V virtual machine resiliency feature from running and therefore pausing virtual machines instead of failing them over to the disaster recovery site, run `(Get-Cluster).ResiliencyDefaultPeriod=0`  

## <a name="FAQ12"></a> How can I reduce time for initial synchronization?  

You can use thin-provisioned storage as one way to speed up initial sync times. Storage Replica queries for and automatically uses thin-provisioned storage, including non-clustered Storage Spaces, Hyper-V dynamic disks, and SAN LUNs.  

You can also use seeded data volumes to reduce bandwidth usage and sometimes time, by ensuring that the destination volume has some subset of data from the primary - via a restored backup, old snapshot, previous replication, copied files, etc. - then using the Seeded option in Failover Cluster Manager or `New-SRPartnership`. If the volume is mostly empty, using seeded sync may reduce time and bandwidth usage.

## <a name="FAQ13"></a> Can I delegate users to administer replication?  

You can use the `Grant-SRDelegation` cmdlet in Windows Server 2016. This allows you to set specific users in server to server, cluster to cluster, and stretch cluster replication scenarios as having the permissions to create, modify, or remove replication, without being a member of the local administrators group. For example:  

    Grant-SRDelegation -UserName contso\tonywang  

The cmdlet will remind you that the user needs to log off and on of the server they are planning to administer in order for the change to take effect. You can use `Get-SRDelegation` and `Revoke-SRDelegation` to further control this.  

## <a name="FAQ13"></a> What are my backup and restore options for replicated volumes?
Storage Replica supports backing up and restoring the source volume. It also supports creating and restoring snapshots of the source volume. You cannot backup or restore the destination volume while protected by Storage Replica, as it is not mounted nor accessible. If you experience a disaster where the source volume is lost, using `Set-SRPartnership` to promote the previous destination volume to now be a read/writable source will allow you to backup or restore that volume. You can also remove replication with `Remove-SRPartnership` and `Remove-SRGroup` to remount that volume as read/writable.
To create periodic application consistent snapshots, you can use VSSADMIN.EXE on the source server to snapshot replicated data volumes. For example, where you are replicating the F: volume with Storage Replica:

    vssadmin create shadow /for=F:
Then, after you switch replication direction, remove replication, or are simply still on the same source volume, you can restore any snapshot to its point in time. For example, still using F:

    vssadmin list shadows
    vssadmin revert shadow /shadow={shadown copy ID GUID listed previously}
You can also schedule this tool to run periodically using a scheduled task. For more information on using VSS, review [Vssadmin](https://technet.microsoft.com/library/cc754968.aspx). There is no need or value in backing up the log volumes. Attempting to do so will be ignored by VSS.
Use of Windows Server Backup, Microsoft Azure Backup, Microsoft DPM, or other snapshot, VSS, virtual machine, or file-based technologies are supported by Storage Replica as long as they operate within the volume layer. Storage Replica does not support block-based backup and restore.

## <a name="FAQ14"></a> Can I configure replication to restrict bandwidth usage?
Yes, via the SMB bandwidth limiter. This is a global setting for all Storage Replica traffic and therefore affects all replication from this server. Typically, this is needed only with Storage Replica initial sync setup, where all the volume data must transfer. If needed after initial sync, your network bandwidth is too low for your IO workload; reduce the IO or increase the bandwidth.

This should only be used with asynchronous replication (note: initial sync is always asynchronous even if you have specified synchronous).
You can also use network QoS policies to shape Storage Replica traffic. Use of highly matched seeded Storage Replica replication will also lower overall initial sync bandwidth usage considerably.


To set the bandwidth limit, use:

    Set-SmbBandwidthLimit  -Category StorageReplication -BytesPerSecond x

To see the bandwidth limit, use:

    Get-SmbBandwidthLimit -Category StorageReplication

To remove the bandwidth limit, use:

    Remove-SmbBandwidthLimit -Category StorageReplication

## <a name="FAQ14"></a> How do I report an issue with Storage Replica or this guide?  
For technical assistance with Storage Replica, you can post at [the Microsoft TechNet forums](https://social.technet.microsoft.com/Forums/windowsserver/en-US/home?forum=WinServerPreview). You can also email srfeed@microsoft.com for questions on Storage Replica or issues with this documentation. The https://windowsserver.uservoice.com site is preferred for design change requests, as it allows your fellow customers to provide support and feedback for your ideas.

## Related Topics  
- [Storage Replica Overview](storage-replica-overview.md) 
- [Stretch Cluster Replication Using Shared Storage](stretch-cluster-replication-using-shared-storage.md)  
- [Server to Server Storage Replication](server-to-server-storage-replication.md)  
- [Cluster to Cluster Storage Replication](cluster-to-cluster-storage-replication.md)  
- [Storage Replica: Known Issues](storage-replica-known-issues.md)  

## See Also  
- [Storage Overview](../storage.md)  
- [Storage Spaces Direct in Windows Server 2016](../storage-spaces/storage-spaces-direct-overview.md)  