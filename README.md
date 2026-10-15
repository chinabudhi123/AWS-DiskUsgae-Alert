# Disk Utilization Monitoring for AWS (Multi-Account, Linux and Windows)

## Goal
Monitor disk usage of all EC2 VMs in all AWS accounts, show it on dashboards (Dev and Prod),
and send an alert when a disk is getting full. A newly created VM must be monitored automatically.

## Tools used
| Need | Tool |
|---|---|
| Inventory and alarm automation | Ansible (playbooks) |
| Account setup in all accounts | CloudFormation StackSet |
| Access to VMs (no SSH/RDP) | AWS Systems Manager (SSM) |
| Agent install and config | SSM State Manager + CloudWatch Agent |
| Auto tag for new VMs | EventBridge + SSM Automation |
| Metrics, dashboards, alarms | CloudWatch |
| Metrics from all accounts in one place | CloudWatch cross-account observability |
| Notification | SNS (email) |

## Why an agent is needed
EBS metrics in CloudWatch show only performance (IOPS, throughput). They do not show how full the disk is.
Disk usage is known only inside the OS, so the CloudWatch Agent is installed on each VM.
Alerts are on the mount level (`/`, `/data`, `D:`), not on the EBS volume.
One LVM mount can use several EBS volumes, so the inventory in Step 1 links mounts to volumes.

## Architecture
- Automation account: Ansible server.
- Member accounts (Dev OU and Prod OU): VMs, SSM, CloudWatch Agent, alarms.
- Monitoring account: dashboards, SNS topic, notification.
- The agent sends metrics to CloudWatch in its own account over HTTPS (443). No network connection to the monitoring account is needed.

## Step 1: Inventory
Find the accounts, regions, VMs, OS and disks.
- Use: Ansible `aws_ec2` inventory plus a playbook that assumes a role in each account.
- It lists: account, region, VM, OS, EBS volumes, SSM managed (yes/no), IAM profile (yes/no), Monitoring tag (yes/no).
- For the mounts, LVM and NFS inside the VM: use an Ansible playbook with the `aws_ssm` connection (works only on SSM-managed VMs).
- Output: CSV files. Review them before moving on.

## Step 2: Create the baseline in every account
- Use: one CloudFormation template deployed as a StackSet to the Dev OU (Environment=Dev) and to the Prod OU (Environment=Prod).
  New accounts in the OU get it automatically.
- The template creates:
  - IAM role `DiskMonAnsibleRole` for Ansible
  - IAM role and instance profile `DiskMonInstanceProfile` for the VMs (policies: `AmazonSSMManagedInstanceCore`, `CloudWatchAgentServerPolicy`)
  - the agent configuration (SSM Parameter Store)
  - the SSM State Manager rules (Step 4)
  - the auto-tag rule (Step 3)
  - the link that shares metrics with the monitoring account

## Step 3: Instance profile and tag
- Tag to use: `Monitoring=enabled`.
- New VMs, profile: use the launch template with `DiskMonInstanceProfile`.
- New VMs, tag: use EventBridge. When a VM starts, it runs an SSM Automation that adds the tag.
- Existing VMs: use an Ansible playbook that attaches `DiskMonInstanceProfile` (if the VM has no profile) and adds the tag.
  Run it in dry-run first, then run it for real.
- VMs that already have another IAM role: add the two policies to that role.
- Check: use the Step 1 playbook again. Rows with "Monitoring tag = no" must be zero.

## Step 4: Agent install
- Use: SSM State Manager. It finds VMs with the tag `Monitoring=enabled`, installs the CloudWatch Agent, and applies the config.
- Linux (Amazon Linux, Ubuntu, RHEL): metric `disk_used_percent` per mount.
- Windows: metric `LogicalDisk % Free Space` per drive.
- Metric namespace: `CWAgent-Dev` for Dev accounts, `CWAgent-Prod` for Prod accounts.
- Check: in CloudWatch Metrics, open the namespace. Each tagged VM should show its mounts within about 10 minutes.
- If no data appears, check in this order: instance profile, SSM shows the VM as Online, the tag, outbound 443 to the SSM and CloudWatch endpoints.

## Step 5: Dashboards
- Use: CloudWatch cross-account observability. Each member account links to the monitoring account.
- Create two dashboards in the monitoring account with CloudFormation: `diskmon-dev` and `diskmon-prod`.
- The split is by namespace (`CWAgent-Dev` and `CWAgent-Prod`), because a dashboard cannot filter on a tag.
- Each dashboard shows the 20 fullest mounts (Linux) and the 20 drives with the least free space (Windows).

## Step 6: Alarms and alerts
- Use: an Ansible playbook that creates the alarms in each account. Run it every hour, so new VMs and new mounts get alarms.
- For every mount it creates two alarms:
  - Warning: used 50% or more (Windows: free 50% or less)
  - Critical: used 80% or more (Windows: free 20% or less)
- The condition must stay true for 3 checks of 5 minutes, to avoid false alerts.
- Alarm name starts with `diskmon-`.
- Alarms send to the SNS topic `diskmon-alerts` in the monitoring account.
- Notification: the SNS topic sends email.

## Step 7: Operations
- Run the Step 1 inventory every week. Goal: zero VMs without tag and zero VMs not managed by SSM.
- New account: put it in the OU (the StackSet deploys the baseline), then add it to the Ansible accounts file.
- New region: deploy the StackSet to that region.

## Where Ansible is used
| Step | Ansible |
|---|---|
| 1 Inventory | Yes, inventory playbooks |
| 3 Tag and profile for existing VMs | Yes, playbook |
| 4 Agent install | No, SSM State Manager does it |
| 6 Alarms | Yes, playbook run every hour |

## Notes
- Test everything in one Dev account first.
- A VM without the tag is not monitored. This is why the check in Step 3 and Step 7 is important.
- If one account has both Dev and Prod VMs, the namespace split will not work. Use separate accounts or a second tag.
