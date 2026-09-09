# RHEL 10.2 Patching POC

Automated patching workflow for RHEL 10.2 VMs using **Red Hat Satellite 6.19**, **Ansible Automation Platform 2.7**, and **Red Hat Lightspeed** in AWS.

## Architecture

This POC is designed for a **disconnected environment**. The connected Satellite is content ingress only. AAP, the downstream Satellite, and all RHEL hosts live in a private subnet with no internet access.

```
                            Internet
                              |
                     Red Hat CDN / registry.redhat.io
                              |
    Connected subnet (10.0.1.0/24)  public + IGW
    +--------------------------------------------------------+
    |  SA laptop / bastion                                   |
    |                                                        |
    |  Connected Satellite 6.19  (RHEL 9, m5.2xlarge)        |
    |    Content ingress only                                |
    |    CV RHEL10: Library -> Dev -> QA -> Prod             |
    |    CV RHEL9-Satellite -> Downstream                    |
    |    AAP 2.7 + RHEL 10 EUS repos synced for ISS         |
    +-----------------------------+--------------------------+
                                  | ISS Network Sync (HTTPS)
    Disconnected subnet (10.0.2.0/24)  VPC-only, no IGW
    +-----------------------------+--------------------------+
    |  AAP 2.7  (RHEL 10.2, m5.xlarge)                      |
    |    Bundle install, no public IP                        |
    |    Registered to disconnected Satellite                |
    |                                                        |
    |  Disconnected Satellite 6.19 + IOP  (RHEL 9, m5.2xl)  |
    |    ISS from connected Satellite                        |
    |    CV RHEL10: Library -> Dev -> QA -> Prod             |
    |    Lightspeed on-prem: Advisor, Vulnerability          |
    |                                                        |
    |  rhel10-dev / rhel10-qa / rhel10-prod  (RHEL 10.2)    |
    |    Registered to disconnected Satellite only           |
    +--------------------------------------------------------+
```



### Disconnected design

This is a **restricted network**, not a physical air gap. The disconnected subnet (`10.0.2.0/24`) has VPC routing to the connected subnet (`10.0.1.0/24`) but no internet gateway. Nothing in the disconnected zone can reach `cdn.redhat.com`, `registry.redhat.io`, `galaxy.ansible.com`, or GitHub.

**Content crosses the boundary in three ways:**

1. **ISS Network Sync** — the disconnected Satellite pulls RPM and errata content from the connected Satellite over HTTPS on the private network. Configured via `hammer organization configure-cdn --type network_sync` in [playbooks/04-deploy-disconnected-satellite.yml](playbooks/04-deploy-disconnected-satellite.yml).
2. **IOP container images** — exported from the connected Satellite with `podman save`, transferred via Ansible `fetch`/`copy` through the bastion, loaded with `podman load` on the disconnected side (playbook 04).
3. **CVE vulnerability data** (`cvemap.xml`) — copied from the connected Satellite to the disconnected Satellite periodically via [playbooks/06-sync-lightspeed-data.yml](playbooks/06-sync-lightspeed-data.yml).



### How disconnected AAP is supplied

AAP never reaches the internet at runtime. Everything it needs is either in the containerized setup bundle or comes from the disconnected Satellite after ISS.


| AAP dependency             | How it is supplied                                                                                                                                                                                                                                                                                      |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Initial install            | AAP 2.7 containerized setup bundle copied via bastion; `bundle_install=true` ([inventory.j2](roles/aap_bootstrap/templates/inventory.j2))                                                                                                                                                               |
| RHEL 10 EUS + AAP 2.7 RPMs | Enable repos on connected Satellite, ISS to disconnected; register AAP with a Satellite activation key (not CDN RHSM)                                                                                                                                                                                   |
| Execution environments     | Default EEs ship inside the setup bundle; custom EEs built on the connected side with `ansible-builder`, `podman save`, transferred in                                                                                                                                                                  |
| Ansible collections        | Downloaded on connected side (`ansible-galaxy collection download`), published into **Private Automation Hub** on AAP (`ansible-galaxy collection publish`), then baked into a custom EE or pulled by Controller at project sync. ISS does **not** carry collections — Private Hub is the supply chain. |
| Playbook project           | Manual/archive upload or a git server inside the VPC. No GitHub.                                                                                                                                                                                                                                        |
| Satellite inventory        | `satellite6` inventory source against the **disconnected** Satellite                                                                                                                                                                                                                                    |
| Insights / CVE data        | IOP on disconnected Satellite; `cvemap.xml` via playbook 05                                                                                                                                                                                                                                             |




### What runs where

- **AAP (disconnected):** Satellite inventory from the downstream Satellite, CV publish/promote on downstream, errata apply to Dev/QA/Prod, host validation, approval gates.
- **SA laptop / connected-side playbooks:** CDN sync on the connected Satellite, ISS trigger, IOP image export, `cvemap.xml` transfer. These jobs need the connected zone and do not belong in the disconnected patching workflow.

Operators reach the AAP UI through the connected Satellite (bastion) or a VPN — AAP has no public IP.

> **Note:** Phase 08 (AAP workflow configuration) requires an SSH local port forward to reach the AAP API from the desktop since AAP has no public IP. See the Quick Start section for the tunnel command.



## Prerequisites

- AWS account with EC2/VPC permissions
- Red Hat subscriptions (AAP, Satellite, RHEL)
- Subscription manifest from [access.redhat.com](https://access.redhat.com)
- Ensure Activation Key has the correct repos added to it. Check through the `inventory/groups_vars/aap.yml` and `satellite.yml` for the repos to add.
- SSH key pair named `patching-poc` in your AWS region with the .pem file downloaded into the root of this repo
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

# Phase 1: Provision VPC, subnets, and routing
ansible-playbook playbooks/01-provision-networking.yml

# Phase 2: Deploy connected Satellite
ansible-playbook playbooks/02-deploy-satellite.yml

# Phase 3: Sync repos on connected Satellite (CDN → upstream)
# Syncs RHEL 10, RHEL 10 EUS, AAP 2.7, RHEL 9 + Satellite repos.
# Before running, check the Satellite GUI subscriptions page and ensure
# the Employee SKU and Satellite Infrastructure Subscription are present.
ansible-playbook playbooks/03-configure-satellite.yml

# Phase 4: Deploy disconnected Satellite with ISS + IOP
# Provisions a RHEL 9 m5.2xlarge EC2 instance in the disconnected subnet,
# installs Satellite, configures ISS Network Sync from the upstream,
# enables/syncs RHEL 10 repos via ISS, exports/loads IOP container images,
# and creates the ak-aap activation key for AAP registration.
ansible-playbook playbooks/04-deploy-disconnected-satellite.yml

# Phase 5: Configure RHEL10 content on disconnected Satellite
# Creates Dev/QA/Prod lifecycle environments, RHEL10 Content View with
# errata filters, publishes and promotes CV versions, creates activation
# keys and host groups. This is where patching content management lives.
ansible-playbook playbooks/05-configure-disconnected-content.yml

# Phase 6: Sync Lightspeed vulnerability data (cvemap.xml)
# Transfers CVE data from connected Satellite to disconnected Satellite.
# Re-run periodically (e.g. weekly) to keep vulnerability data current.
ansible-playbook playbooks/06-sync-lightspeed-data.yml

# Phase 7: Deploy AAP 2.7 Controller in the disconnected subnet
# AAP has no public IP; SSH via ProxyJump through the connected Satellite.
# Registered to the disconnected Satellite (not CDN).
ansible-playbook playbooks/07-deploy-aap.yml

# Phase 8: Provision RHEL 10.2 VMs in disconnected subnet
# VMs register to the disconnected Satellite, not the CDN
ansible-playbook playbooks/08-provision-vms.yml

# Phase 9: Configure AAP Workflow
# Before running, open an SSH tunnel for the AAP API:
#   ssh -f -N -L 8443:<aap_private_ip>:443 ec2-user@<satellite_public_ip>
ansible-playbook playbooks/09-configure-aap-workflow.yml
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
ansible-playbook playbooks/06-sync-lightspeed-data.yml
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
│   ├── 05-configure-disconnected-content.yml
│   ├── 06-sync-lightspeed-data.yml
│   ├── 07-deploy-aap.yml
│   ├── 08-provision-vms.yml
│   ├── 09-configure-aap-workflow.yml
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


