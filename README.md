# RHEL 10.2 Patching POC

Automated patching workflow for RHEL 10.2 VMs using **Red Hat Satellite 6.19**, **Ansible Automation Platform 2.7**, and **Red Hat Lightspeed** in AWS.

## Architecture

```
                         Connected Subnet (10.0.1.0/24)
                         ┌─────────────────────────────────────────┐
Desktop (Ansible) ──────>│ AAP 2.7 (RHEL 10.2, m5.xlarge)         │
                         │                                         │
  Red Hat CDN ──────────>│ Upstream Satellite 6.19 (RHEL 9, m5.2xl)│
                         │   +-- Content View: RHEL10              │
                         │   |     Library -> Dev -> QA -> Prod    │
                         │   +-- Content View: RHEL9-Satellite     │
                         │   |     Library -> Downstream           │
                         └───────────────┬─────────────────────────┘
                                         │ ISS Network Sync
                         Disconnected Subnet (10.0.2.0/24)
                         ┌───────────────┴─────────────────────────┐
                         │ Downstream Satellite 6.19 + IOP         │
                         │   (RHEL 9, m5.2xlarge)                  │
                         │   +-- ISS from upstream Satellite       │
                         │   +-- Content View: RHEL10              │
                         │   +-- Lightspeed: Advisor, Vulnerability│
                         │                                         │
                         │ rhel10-dev  (RHEL 10.2, t3.medium)      │
                         │ rhel10-qa   (RHEL 10.2, t3.medium)      │
                         │ rhel10-prod (RHEL 10.2, t3.medium)      │
                         └─────────────────────────────────────────┘
                         (No internet access - VPC-only routing)
```



## Prerequisites

- AWS account with EC2/VPC permissions
- Red Hat subscriptions (AAP, Satellite, RHEL)
- Subscription manifest from [access.redhat.com](https://access.redhat.com)
- Ensure Activation Key has the correct repos added to it. Check through the `inventory/groups_vars/aap.yml` and `satellite.yml` for the repos to add.
- SSH key pair named `patching-poc` in your AWS region
- Ansible Core 2.16+ installed locally
- Vault password file for encrypted credentials
- Red Hat Automation Hub API token (see below)
- Download the latest AAP bundle here: [https://access.redhat.com/downloads/content/480/ver=2.7/rhel---10/2.7/x86_64/product-software](https://access.redhat.com/downloads/content/480/ver=2.7/rhel---10/2.7/x86_64/product-software) and set its location in `credentials.yml`



### Automation Hub Token

The `redhat.satellite`, `redhat.satellite_operations`, and `ansible.platform` collections
are only available from Red Hat Automation Hub (not public Galaxy). The `ansible.cfg` in this
project is pre-configured to use Automation Hub as the primary source.

1. Go to [console.redhat.com/ansible/automation-hub/token](https://console.redhat.com/ansible/automation-hub/token/)
2. Click **Load Token** and copy the generated token
3. Place the token in the `credentials.yml` file for variable `automation_hub_token`

Also, set the environment variable:

```bash
export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN="your-token-here"
```



## Quick Start

```bash
# Install required collections
ansible-galaxy collection install -r requirements.yml

# Configure AWS credentials
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="ap-southeast-2"

# Phase 1: Deploy AAP Controller
ansible-playbook playbooks/01-deploy-aap.yml

# Phase 2: Deploy Satellite (run from AAP or locally)
ansible-playbook playbooks/02-deploy-satellite.yml

# Phase 3: Configure Satellite content
# Before running the next playbook double check the subscriptions have been added to the Satellite instance by going to the GUI at the public ip of the instance -> selecting the subscriptions page -> check the Employee SKU and the Red Hat Satellite Infrastructure Subscription have been added. If not click Add Subscriptions and add 1 entitlement of each
ansible-playbook playbooks/03-configure-satellite.yml

# Phase 4: Deploy disconnected Satellite Server with IOP
# Creates a private subnet, provisions a RHEL 9 m5.2xlarge EC2 instance,
# installs Satellite, configures ISS Network Sync from the upstream,
# exports/loads IOP container images, and enables Lightspeed on-prem.
ansible-playbook playbooks/04-deploy-disconnected-satellite.yml

# Phase 5: Sync Lightspeed vulnerability data (cvemap.xml)
# Transfers CVE data from connected Satellite to disconnected Satellite.
# Re-run periodically (e.g. weekly) to keep vulnerability data current.
ansible-playbook playbooks/05-sync-lightspeed-data.yml

# Phase 6: Provision RHEL 10.2 VMs in disconnected subnet
# VMs register to the disconnected Satellite, not the CDN
ansible-playbook playbooks/06-provision-vms.yml

# Phase 7: Configure AAP Workflow
ansible-playbook playbooks/07-patching-workflow.yml
```



## Credential Variables

Copy the sample credentials.yml.example to credentials.yml and enter your details:

```yaml
# Red Hat Subscription Manager
rhsm_org_id: "CHANGE_ME"
rhsm_activation_key: "CHANGE_ME"

# AAP 2.7
aap_admin_password: "CHANGE_ME"
aap_pg_password: "CHANGE_ME"
aap_setup_bundle_path: "~/Downloads/ansible-automation-platform-containerized-setup-bundle-2.7-x86_64.tar.gz"
registry_username: ""
registry_password: ""

# Satellite 6.19
satellite_admin_password: "CHANGE_ME"
satellite_manifest_path: "/tmp/manifest.zip"

# AWS
aws_key_name: patching-poc


```



## Patching Workflow

The AAP Workflow Template "RHEL 10 Patching Cadence" executes:

1. Sync repos and publish new Content View version
2. Promote to Dev -> Apply errata to Dev hosts -> Validate
3. **Approval gate** (manual)
4. Promote to QA -> Apply errata to QA hosts -> Validate
5. **Approval gate** (manual)
6. Promote to Prod -> Apply errata to Prod hosts -> Validate

Scheduled to run weekly on Tuesdays at 06:00 UTC.

## Red Hat Lightspeed Integration

Lightspeed on-prem (IOP) runs on the **disconnected Satellite Server** as 19 containerized
microservices. It provides Advisor and Vulnerability intelligence locally without sending
data outside your environment. IOP container images are exported from the connected
Satellite and loaded on the disconnected side. CVE vulnerability data (`cvemap.xml`) is
synced periodically from the connected Satellite using playbook 05.

Sync vulnerability data:

```bash
ansible-playbook playbooks/05-sync-lightspeed-data.yml
```



### Vulnerability Intelligence

Use the Lightspeed Vulnerability MCP to identify CVEs affecting managed hosts:

```
- vulnerability__get_cves: List CVEs affecting your fleet
- vulnerability__get_cve_systems: Find which systems are affected by a specific CVE
- vulnerability__get_system_cves: List all CVEs for a specific system
- vulnerability__explain_cves: Get explanation of why CVEs affect your environment
```



### Advisor Recommendations

Proactive system health via Lightspeed Advisor:

```
- advisor__get_active_rules: Get active recommendations for your systems
- advisor__get_rule_details: Deep-dive into a specific recommendation
- advisor__get_hosts_hitting_a_rule: Find affected hosts
```



### Satellite Content Operations (via MCP)

Direct content lifecycle management through Lightspeed:

```
- publish_content_view: Publish new CV version
- promote_content_view_version: Promote through lifecycle environments
- incremental_content_view_update: Targeted errata addition to CV versions
- trigger_remote_execution_job: Apply errata via Satellite remote execution
```



### Planning and Lifecycle

Stay ahead of RHEL changes:

```
- planning__get_upcoming_changes: Upcoming deprecations and enhancements
- planning__get_rhel_lifecycle: RHEL version support timelines
- planning__get_relevant_upcoming: Changes relevant to your registered systems
```



### Remediation Playbooks

Lightspeed can generate Ansible remediation playbooks for specific CVEs,
which can be imported into AAP as Job Templates for targeted remediation
outside the regular patching cadence.

## File Structure

```
Patching-POC/
├── ansible.cfg
├── requirements.yml
├── README.md
├── credentials.yml
├── inventory/
│   ├── aws_ec2.yml
│   └── group_vars/
│       ├── all.yml
│       ├── aap.yml
│       ├── satellite.yml
│       └── disconnected_satellite.yml
├── playbooks/
│   ├── 01-deploy-aap.yml
│   ├── 02-deploy-satellite.yml
│   ├── 03-configure-satellite.yml
│   ├── 04-deploy-disconnected-satellite.yml
│   ├── 05-sync-lightspeed-data.yml
│   ├── 06-provision-vms.yml
│   ├── 07-patching-workflow.yml
│   ├── publish_promote_downstream.yml
│   ├── patch-publish-cv.yml
│   ├── patch-promote.yml
│   ├── patch-apply.yml
│   ├── patch-validate.yml
│   └── vars/
│       ├── aws.yml
│       ├── aws_resources.yml                     (auto-generated by 01)
│       ├── satellite_resources.yml               (auto-generated by 02)
│       └── disconnected_satellite_resources.yml  (auto-generated by 04)
├── roles/
│   ├── aap_bootstrap/
│   ├── satellite_deploy/
│   ├── satellite_configure/
│   └── satellite_disconnected_deploy/
└── templates/
```



## AWS Cost Estimate


| Component                | Instance   | Storage     | Monthly   |
| ------------------------ | ---------- | ----------- | --------- |
| AAP 2.7                  | m5.xlarge  | 100GB gp3   | ~$140     |
| Upstream Satellite 6.19  | m5.2xlarge | 500GB gp3   | ~$310     |
| Downstream Satellite+IOP | m5.2xlarge | 500GB gp3   | ~$310     |
| RHEL 10.2 VMs (x3)       | t3.medium  | 30GB gp3 ea | ~$90      |
| **Total**                |            |             | **~$850** |


