# Multi-account disk monitoring on AWS (Ansible + SSM + CloudWatch)

Goal: detect low disk space on every EC2 VM across every AWS account/region early, using the
existing Ansible stack plus only the cloud-native services that clearly earn their place
(SSM, CloudWatch, Organizations/StackSets). No third-party monitoring platform, no SSH, no inbound ports.

## Architecture

```
 Automation account                    Member accounts (1..N)                     Monitoring account
 ┌───────────────────────┐ AssumeRole ┌──────────────────────────────┐  metrics   ┌──────────────────────┐
 │ Ansible control node  │──────────▶ │ DiskMonAnsibleRole           │ ─(OAM)───▶ │ OAM sink             │
 │ aws_ec2 inventory     │ SSM session│ EC2 + SSM agent + CW agent   │            │ CloudWatch dashboard │
 │ CloudFormation StackSets│─deploy──▶│ SSM State Manager (enrol all)│  alarms    │ SNS diskmon-alerts   │
 └───────────────────────┘            │ Alarms diskmon-warn/crit-*   │ ─(SNS)───▶ │ -> Slack / ITSM      │
                                      └──────────────────────────────┘            └──────────────────────┘
```

Two collection paths, deliberately:

| Path | Purpose | Mechanism |
|---|---|---|
| Continuous (primary) | Early warning, trends, alerting | CloudWatch agent publishes `disk_used_percent` every 5 min; alarms at 80% (warn) and 90% (crit), 3 consecutive periods |
| On-demand (Ansible) | Audit, coverage proof, incident triage | `02_disk_report.yml` runs over SSM, writes `reports/disk_report.csv`, lists VMs Ansible could not reach |

## 1. Access management

- **Cross-account**: one role per account, `DiskMonAnsibleRole`, deployed by an Organizations service-managed
  StackSet (`cloudformation/member-global.yaml`). Trust is limited to the single control-node role, not the account root.
  The control node chains into each account with `credential_source = Ec2InstanceMetadata`
  (profiles generated into `generated/aws_config`). No long-lived keys anywhere.
- **Least privilege**: describe/list; `ssm:StartSession` only on instances in that account and not tagged `diskmon=disabled`;
  S3 only on `ansible-ssm-<acct>-*`; alarm writes only on `alarm:diskmon-*`.
- **Connectivity**: Ansible uses the `community.aws.aws_ssm` connection plugin: Session Manager over HTTPS,
  so no SSH keys, no port 22, no bastions, works for private subnets (needs SSM VPC endpoints or NAT).
  Sessions are IAM-authenticated and CloudTrail-logged; enable Session Manager logging to S3/CloudWatch for command audit.
- **VM-side**: Default Host Management Configuration (DHMC, `01_bootstrap_ssm.yml`) makes every instance SSM-managed
  without instance-profile work; `DiskMonInstanceRole` adds `CloudWatchAgentServerPolicy` for metric publishing.

## 2. Data collection and aggregation

- **Collection**: SSM State Manager associations (`member-regional.yaml`) target `InstanceIds: *`, so every managed instance
  automatically gets the CloudWatch agent installed and configured from SSM Parameter `/diskmon/cwagent-config`.
  Re-applied daily to correct drift. Ansible role `cw_agent` is the equivalent fallback (`03_install_agent.yml`, rolling 20%).
- **Aggregation**: CloudWatch cross-account observability (OAM) links each member account/region to the monitoring account,
  so one dashboard (`diskmon-fleet`) shows the fullest filesystems fleet-wide via Metrics Insights queries.
- **Alerting**: `04_reconcile_alarms.yml` creates a warn and a crit alarm per (instance, mount) that is publishing data;
  alarms notify the central SNS topic (email now; Slack via AWS Chatbot, ITSM via Lambda/EventBridge later).
- **Ansible report**: `02_disk_report.yml` (agentless) for audits.

## 3. Scalability and enrolment

| Event | What happens | Manual work |
|---|---|---|
| New VM | DHMC registers it; State Manager installs/configures agent; next hourly reconcile adds alarms | none |
| New volume/mount | agent publishes new `path` dimension; reconcile adds alarms | none |
| New account | joins the Org OU -> StackSet auto-deploys roles/plumbing; add to `accounts.yml` (or `discover_from_org: true`) and run `00_generate_inventory.yml` | one command |
| New region | add to `regions`; deploy `member-regional.yaml` there | one StackSet region |
| VM opt-out | tag `diskmon=disabled` | |
| Coverage gaps | `02_disk_report.yml` lists UNREACHABLE instances (not SSM-managed) | investigate |

Inventory is one `aws_ec2` source per account (parallel, cached), `forks: 50`, `strategy: free`.
Roll-outs are `serial: 20%`.

## Deploy order

1. Monitoring account: `cloudformation/monitoring-account.yaml` (per region) -> note `OamSinkArn`, `AlertTopicArn`.
2. Management/delegated-admin account: StackSets, service-managed, auto-deploy to the target OUs:
   `member-global.yaml` (one region) and `member-regional.yaml` (each region; pass `OamSinkArn`).
3. Control node: `ansible-galaxy collection install -r requirements.yml`, install `boto3`, AWS CLI and the Session Manager plugin.
4. Edit `accounts.yml` and `group_vars/all.yml` (`alarm_topic_arn_template`).
5. `ansible-playbook playbooks/00_generate_inventory.yml`
6. `scripts/run.sh playbooks/01_bootstrap_ssm.yml`
7. `scripts/run.sh playbooks/02_disk_report.yml` (smoke test: should list your VMs and CSV)
8. `scripts/run.sh playbooks/04_reconcile_alarms.yml` (schedule hourly)

## Known limits and trade-offs

- Windows is alerted on FREE space (`LogicalDisk % Free Space`), Linux on USED space; thresholds are kept consistent (80% used == 20% free).
- Per-instance alarm reconcile is simple and explicit but is O(metrics) API calls. Around several thousand
  filesystems, move to CloudWatch Metrics Insights multi-series alarms or a Lambda reconciler.
- CloudWatch cost is per metric (about one per instance per mount per month). Collect only `used_percent`; fewer mounts via `resources`.
- Disk fill rate matters more than level for fast-filling volumes: add a `PREDICT_LINEAR`/anomaly alarm later.
- Alarm SNS topic must be in the same region as the alarm (one topic per region).
- The StackSet/OAM/SSM-document details were written to AWS docs but not deployed in a live Org from this repo;
  run `cfn-lint` and a pilot OU first.

## Operating system support

| OS | Agent install (State Manager) | Metric | Alarm | Ansible fallback |
|---|---|---|---|---|
| Amazon Linux 2 / 2023 | `AWS-ConfigureAWSPackage` | `disk_used_percent` (dim: InstanceId, path, device, fstype) | used >= 80 / 90 | `roles/cw_agent/tasks/linux.yml` (package) |
| Ubuntu / Debian | same | same | same | `.deb` from AWS |
| RHEL / CentOS / Rocky | same | same | same | `.rpm` from AWS |
| Windows Server | same | `LogicalDisk % Free Space` (dim: InstanceId, instance e.g. `C:`) | free <= 20 / 10 | `roles/cw_agent/tasks/windows.yml` (MSI) |

How one rollout serves both OS families: the `Diskmon-ConfigureAgent` SSM document has one step per platform
(precondition `platformType`), each applying its own config parameter
(`/diskmon/cwagent-config-linux` or `/diskmon/cwagent-config-windows`). Ansible groups hosts into
`linux` / `windows` from EC2 platform details; PowerShell shell type is set for Windows via `inventory/group_vars/windows.yml`.
Windows over SSM needs `ansible.windows` and a PowerShell-capable SSM agent (current AMIs have it).
