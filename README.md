# Azure Backup & Disaster Recovery Lab – File-Level and Full VM Recovery

## Overview

This lab demonstrates a complete Azure Virtual Machine backup and disaster recovery workflow using **Azure Backup** and a **Recovery Services Vault**.

The goal was not simply to configure a backup and confirm that a backup job completed. I intentionally simulated two different recovery scenarios:

1. Accidental deletion of application data requiring **file-level recovery**
2. Complete loss of the virtual machine requiring **full VM recovery**

After restoring the VM, I also troubleshot network and authentication issues before verifying that the application data survived the recovery process.

---

# Lab Objectives

The lab covered:

- Deploying a Linux virtual machine
- Creating sample application/customer data
- Creating a Recovery Services Vault
- Creating and assigning an Azure VM backup policy
- Running an on-demand backup
- Verifying a recovery point
- Simulating accidental file deletion
- Performing Azure File Recovery
- Mounting a recovery point using Microsoft's recovery script
- Recovering an individual file
- Unmounting the recovery disks
- Simulating complete VM loss
- Restoring the entire VM from Azure Backup
- Using a staging storage account during VM recovery
- Troubleshooting SSH connectivity after restoration
- Configuring an NSG to permit SSH
- Resetting/restoring VM authentication
- Connecting to the restored VM
- Verifying recovered application data

---

# Architecture

```text
                    Azure Subscription
                           |
                           |
                    Resource Group
                           |
              +------------+------------+
              |                         |
              |                         |
           Azure VM              Recovery Services Vault
     vnet-backup-lab01            rsv-iqra-prod-01
              |                         |
              |                         |
        Ubuntu Linux               Backup Policy
              |                   pol-vm-prod-4h
              |                         |
       /srv/iqras-app/                  |
              |                         |
       customer-data/              Recovery Point
              |                         |
        customers.csv                   |
                                        |
                           +------------+------------+
                           |                         |
                     File Recovery              VM Recovery
                           |                         |
                    Mounted recovery        Restored Azure VM
                         disks            vnet-backup-lab01-restored
                           |                         |
                    customers.csv              Data verified
```

---

# Environment

| Component | Purpose |
|---|---|
| Azure Virtual Machine | Protected Linux workload |
| Ubuntu Linux | VM operating system |
| Recovery Services Vault | Azure Backup management |
| Azure Backup Policy | Backup schedule and retention |
| Azure Recovery Point | Point-in-time recovery source |
| Azure Storage Account | Temporary staging location during VM restore |
| Azure VNet/Subnet | VM network connectivity |
| Network Security Group | Network access control |
| SSH | Linux administration and validation |

---

# 1. Create the Application Data

The VM represented a small production application server.

Application directories were created under:

```bash
/srv/iqras-app/
```

Example structure:

```text
/srv/iqras-app/
├── app.log
├── config
├── customer-data
└── reports
```

The customer database simulation was stored at:

```text
/srv/iqras-app/customer-data/customers.csv
```

Example data:

```csv
customer_id,name,tier
1001,Ali,Premium
1002,Umer,Standard
1003,Usman,Premium
1004,Hamdan,Standard
```

Verify the file:

```bash
cat /srv/iqras-app/customer-data/customers.csv
```

The `/srv` application directory was assigned appropriate ownership for the lab:

```bash
sudo chown -R azureuser:azureuser /srv/iqras-app
```

---

# 2. Create a Recovery Services Vault

A Recovery Services Vault was created:

```text
rsv-iqra-prod-01
```

The vault acts as the management layer for Azure VM backups and recovery points.

The protected workload was:

```text
vnet-backup-lab01
```

---

# 3. Configure the VM Backup Policy

A custom Azure VM backup policy was created:

```text
pol-vm-prod-4h
```

The policy controlled:

- Backup scheduling
- Recovery point creation
- Retention
- VM protection

The VM was then registered with the Recovery Services Vault and protected using the policy.

---

# 4. Run the Initial Backup

Instead of waiting for the normal backup schedule, an on-demand backup was started.

The backup job was monitored from:

```text
Recovery Services Vault
    ↓
Backup Jobs
```

The job completed successfully.

A recovery point was then available for the VM.

Example recovery point used during the lab:

```text
09/19/2026 09:13 AM
```

At this point Azure contained a recoverable copy of the VM.

---

# 5. Simulate Accidental File Deletion

The first failure scenario simulated a user or administrator accidentally deleting production data.

The customer file was intentionally removed:

```bash
sudo rm /srv/iqras-app/customer-data/customers.csv
```

Verify the deletion:

```bash
ls -l /srv/iqras-app/customer-data/
```

Attempting to read the file now failed:

```bash
cat /srv/iqras-app/customer-data/customers.csv
```

Expected result:

```text
No such file or directory
```

The VM itself was still healthy.

Only application data had been lost.

This is a good use case for **file-level recovery instead of restoring an entire VM**.

---

# 6. Start Azure File Recovery

From the protected VM inside the Recovery Services Vault:

```text
Backup Item
    ↓
File Recovery
```

The required recovery point was selected.

Azure generated a recovery script and temporary password.

The script was downloaded to the administration workstation.

---

# 7. Transfer the Recovery Script to the VM

The Microsoft recovery script was transferred from the local Linux workstation to the Azure VM.

Example:

```bash
scp ~/Downloads/largedisk_0_vnet-backup-lab01*.py azureuser@<VM-PUBLIC-IP>:/home/azureuser/
```

SSH into the VM:

```bash
ssh azureuser@<VM-PUBLIC-IP>
```

Verify the script:

```bash
ls
```

---

# 8. Run the Azure File Recovery Script

The recovery script was executed with elevated privileges:

```bash
sudo python3 largedisk_0_vnet-backup-lab01*.py
```

During execution the script checked the operating system and required packages.

The system required:

```text
setfacl
```

Azure's script installed the required ACL package.

Because the VM was using a newer Python release, another compatibility issue appeared.

The recovery utility required the Python:

```text
asyncore
```

module.

`asyncore` is no longer available in newer Python versions.

The recovery script detected the issue and offered to install:

```text
pyasyncore
```

The installation was accepted.

After installation:

```text
asyncore is now available - SecureTCPTunnel can proceed
```

This was an important troubleshooting step during the recovery process.

---

# 9. Connect to the Recovery Point

The temporary password displayed in the Azure portal was entered into the recovery script.

Azure established an iSCSI connection to the recovery point.

The script reported:

```text
Connection succeeded!
```

Azure then attached the recovery-point volumes to the VM.

Example mounted volumes:

```text
/dev/sdc1
/dev/sdc15
/dev/sdc16
```

One BIOS boot partition was not mounted because it did not contain a normal filesystem.

This was expected and was not required for recovering the application file.

The primary recovered filesystem was mounted under a path similar to:

```text
/home/azureuser/vnet-backup-lab01-<timestamp>/Volume1
```

---

# 10. Locate the Deleted File Inside the Recovery Point

The recovered application directory was inspected:

```bash
sudo ls -l /home/azureuser/vnet-backup-lab01-<timestamp>/Volume1/srv/iqras-app/customer-data/
```

The deleted file was present:

```text
customers.csv
```

The contents were verified directly from the recovery point:

```bash
sudo cat /home/azureuser/vnet-backup-lab01-<timestamp>/Volume1/srv/iqras-app/customer-data/customers.csv
```

Output:

```csv
customer_id,name,tier
1001,Ali,Premium
1002,Umer,Standard
1003,Usman,Premium
1004,Hamdan,Standard
```

This confirmed that the Azure recovery point contained the lost application data.

---

# 11. Restore the Individual File

The file was copied from the mounted recovery point back into the production directory.

```bash
sudo cp /home/azureuser/vnet-backup-lab01-<timestamp>/Volume1/srv/iqras-app/customer-data/customers.csv /srv/iqras-app/customer-data/customers.csv
```

Verify the restored file:

```bash
sudo cat /srv/iqras-app/customer-data/customers.csv
```

Output:

```csv
customer_id,name,tier
1001,Ali,Premium
1002,Umer,Standard
1003,Usman,Premium
1004,Hamdan,Standard
```

The file-level recovery was successful.

---

# 12. Unmount the Recovery Disks

After recovering the required file, the recovery-point disks were disconnected.

From Azure:

```text
File Recovery
    ↓
Unmount Disks
```

Azure reported:

```text
Unmount successful
```

The recovery session was now closed.

This is important because recovery disks should not remain unnecessarily mounted after the recovery operation.

---

# 13. Simulate Complete VM Failure

The next scenario represented a much larger incident.

Instead of deleting one file, the entire protected VM was deleted.

The VM:

```text
vnet-backup-lab01
```

was removed from Azure.

The VM and selected associated resources were deleted.

At this point the production virtual machine no longer existed.

However, its recovery point remained protected inside the Recovery Services Vault.

---

# 14. Start Full VM Recovery

The protected backup item was opened from the Recovery Services Vault.

The VM restore workflow was started.

Restore target:

```text
Create new
```

Restore type:

```text
Create new virtual machine
```

New VM name:

```text
vnet-backup-lab01-restored
```

The VM was restored into the existing lab resource group and virtual network.

---

# 15. Configure the Staging Storage Account

Azure required a temporary staging storage account during the restore operation.

A new storage account was created:

```text
stiqrabackup
```

Configuration included:

```text
Region: East US
Performance: Standard
Redundancy: LRS
```

The storage account was then selected as the:

```text
Staging Location
```

for the restore job.

The staging account is used by Azure Backup during parts of the VM recovery workflow.

It is not the permanent location of the restored VM.

---

# 16. Monitor the VM Restore Job

The restore was monitored from:

```text
Recovery Services Vault
    ↓
Backup Jobs
    ↓
Restore
```

The restore job showed:

```text
Job Type:
Recover VM to an alternate location
```

Original VM:

```text
vnet-backup-lab01
```

Target VM:

```text
vnet-backup-lab01-restored
```

Subtasks included:

```text
Transfer data from vault
Create the restored virtual machine
```

Azure eventually recreated the VM successfully.

The restored VM appeared in:

```text
Azure Portal
    ↓
Virtual Machines
```

as:

```text
vnet-backup-lab01-restored
```

---

# 17. Initial SSH Failure After VM Recovery

After the VM was restored, I attempted to connect using SSH:

```bash
ssh azureuser@<RESTORED-VM-PUBLIC-IP>
```

The initial connection timed out:

```text
ssh: connect to host <RESTORED-VM-PUBLIC-IP> port 22: Connection timed out
```

This indicated a network access problem rather than a failed VM recovery.

The VM was running and had a public IP, but inbound SSH was not currently permitted.

---

# 18. Troubleshoot the Restored VM Network

The restored VM's networking configuration was inspected.

The restored VM had:

- A new network interface
- A new public IP address
- Connectivity to the existing VNet/subnet
- No effective SSH rule permitting TCP/22

The VM's Network Security Group configuration therefore had to be corrected.

An NSG was associated/configured with an inbound rule allowing:

```text
Protocol: TCP
Destination Port: 22
Action: Allow
```

After the NSG was applied, SSH was attempted again:

```bash
ssh azureuser@<RESTORED-VM-PUBLIC-IP>
```

This time the VM responded.

The SSH client displayed the server fingerprint prompt:

```text
The authenticity of host '<RESTORED-VM-PUBLIC-IP>' can't be established.
```

After accepting the fingerprint, the connection reached the authentication stage.

This proved that the network connectivity issue was fixed.

---

# 19. Authentication Failure

Although TCP/22 was now reachable, authentication initially failed:

```text
Permission denied, please try again.
```

This demonstrated an important distinction:

```text
Connection timeout
        ↓
Network / NSG problem

Permission denied
        ↓
Authentication / credential problem
```

The restored operating system contained the backed-up Linux user configuration, but valid authentication still needed to be established for administrative access.

The Azure VM password reset functionality was used to reset the `azureuser` credentials.

After resetting the credentials, SSH was attempted again.

```bash
ssh azureuser@<RESTORED-VM-PUBLIC-IP>
```

The login succeeded.

---

# 20. Verify the Restored VM

After connecting to the restored VM, the original application directory was checked.

```bash
cd /srv
```

```bash
ls
```

The application directory existed:

```text
iqras-app
```

Navigate into it:

```bash
cd /srv/iqras-app
```

Verify the application structure:

```bash
ls
```

The original directories and files were present.

Navigate to the customer data:

```bash
cd customer-data
```

```bash
ls
```

The recovered VM contained:

```text
customers.csv
```

Finally, the data was verified:

```bash
cat customers.csv
```

Output:

```csv
customer_id,name,tier
1001,Ali,Premium
1002,Umer,Standard
1003,Usman,Premium
1004,Hamdan,Standard
```

This confirmed that the complete VM recovery was successful.

---

# Troubleshooting Summary

Several real-world issues occurred during the lab.

## Issue 1 – Python `asyncore` Compatibility

### Problem

The Azure File Recovery script required the Python `asyncore` module, which was unavailable in the newer Python environment.

### Resolution

The script detected the problem and installed:

```text
pyasyncore
```

This allowed the Secure TCP Tunnel component to continue.

### Lesson

Recovery tooling can have dependencies on the operating system and runtime environment. Backup administrators should test recovery procedures before an actual incident.

---

## Issue 2 – BIOS Boot Partition Did Not Mount

Azure File Recovery reported that one partition could not be mounted:

```text
BIOS Boot partition
```

This was not required for the file recovery operation.

The filesystem containing `/srv/iqras-app` mounted correctly and contained the required application data.

### Lesson

Not every attached partition represents user-accessible application data.

---

## Issue 3 – Permission Denied Accessing Recovery Directory

The recovery script created its mount directory with permissions that prevented normal traversal as `azureuser`.

Attempting to enter the recovery directory directly resulted in:

```text
Permission denied
```

The recovery files were therefore accessed using elevated privileges:

```bash
sudo ls
sudo cat
sudo cp
```

### Lesson

Mounted recovery data may require administrative privileges even when the original application directory was owned by the application user.

---

## Issue 4 – SSH Timed Out After Full VM Restore

### Problem

```text
ssh: connect to host <IP> port 22: Connection timed out
```

### Investigation

The restored VM was:

- Running
- Assigned a public IP
- Connected to the VNet
- Missing effective inbound SSH access

### Resolution

An NSG was configured/associated to allow inbound TCP port 22.

### Lesson

Restoring the compute workload does not automatically mean every external connectivity dependency will behave exactly like the original environment.

Recovery validation must include networking.

---

## Issue 5 – SSH Reached VM but Authentication Failed

After fixing the NSG:

```text
Permission denied, please try again.
```

This proved the network problem was solved because the SSH daemon was now reachable.

The remaining issue was authentication.

### Resolution

The `azureuser` credentials were reset through Azure's VM recovery/reset functionality.

SSH then succeeded.

### Lesson

Troubleshooting should separate:

```text
Network connectivity
        ↓
Transport/service availability
        ↓
Authentication
        ↓
Application validation
```

Each layer should be verified independently.

---

# Disaster Recovery Validation

The lab validated two different recovery objectives.

| Failure | Recovery Method | Result |
|---|---|---|
| Individual file deleted | Azure File Recovery | Successful |
| Entire VM deleted | Azure VM Restore | Successful |
| SSH unavailable after restore | NSG troubleshooting | Resolved |
| Authentication failure | Credential reset | Resolved |
| Application data verification | Linux CLI | Successful |

---

# Recovery Workflow

```text
Create Linux VM
      ↓
Create application data
      ↓
Configure Recovery Services Vault
      ↓
Configure backup policy
      ↓
Run backup
      ↓
Create recovery point
      ↓
Delete customers.csv
      ↓
Run Azure File Recovery
      ↓
Mount recovery point
      ↓
Locate customers.csv
      ↓
Copy file back to production
      ↓
Verify recovered file
      ↓
Unmount recovery disks
      ↓
Delete entire VM
      ↓
Select recovery point
      ↓
Create staging storage account
      ↓
Restore VM to alternate location
      ↓
VM recreated
      ↓
SSH timeout
      ↓
Investigate networking
      ↓
Configure NSG TCP/22
      ↓
SSH reaches VM
      ↓
Authentication failure
      ↓
Reset credentials
      ↓
SSH successful
      ↓
Verify /srv/iqras-app
      ↓
Verify customers.csv
      ↓
DISASTER RECOVERY VALIDATED
```

---

# Key Lessons

## Backup Is Not the Same as Recovery

A successful backup job only confirms that Azure created a recovery point.

The real test is whether the workload can actually be recovered.

This lab tested both:

```text
Backup success
```

and:

```text
Recovery success
```

---

## Use the Smallest Appropriate Recovery Method

Deleting one file does not require restoring an entire VM.

Azure File Recovery allowed the required file to be recovered while the production VM remained online.

Full VM recovery was reserved for the simulated complete VM-loss scenario.

---

## Recovery Must Include Validation

A VM appearing as `Running` in Azure does not mean the recovery process is finished.

The restored workload still required validation of:

- Networking
- NSG rules
- SSH connectivity
- Authentication
- Filesystem
- Application directories
- Application data

Only after these checks was the recovery considered successful.

---

## Recovery Procedures Should Be Tested

This lab exposed issues that would not have been discovered by simply configuring Azure Backup:

- Python compatibility
- Recovery mount permissions
- Recovery disk behavior
- Staging storage requirements
- Restored networking differences
- NSG configuration
- Authentication recovery

Testing recovery procedures before a real outage helps identify these dependencies ahead of time.

---

# Skills Practiced

- Microsoft Azure
- Azure Backup
- Recovery Services Vault
- Azure VM Backup
- Backup policies
- Recovery points
- Azure File Recovery
- Full VM disaster recovery
- Azure Storage Accounts
- Azure Virtual Networks
- Network Security Groups
- Linux administration
- SSH
- SCP
- Linux filesystem permissions
- Recovery troubleshooting
- Incident recovery validation

---

# Final Result

The lab successfully demonstrated an end-to-end backup and disaster recovery workflow.

The first incident simulated accidental deletion of production application data. The individual file was recovered from an Azure recovery point without rebuilding the VM.

The second incident simulated complete loss of the production virtual machine. The VM was deleted and recreated from Azure Backup using a recovery point.

After restoration, network and authentication issues were diagnosed and corrected. SSH access was restored and the original application data was verified on the recovered VM.

Final validation:

```bash
cat /srv/iqras-app/customer-data/customers.csv
```

```text
customer_id,name,tier
1001,Ali,Premium
1002,Umer,Standard
1003,Usman,Premium
1004,Hamdan,Standard
```

**Result: File-level recovery and complete VM disaster recovery successfully validated.**

---

The next backup and disaster recovery lab will focus on protecting and recovering database workloads using Azure SQL recovery capabilities.
