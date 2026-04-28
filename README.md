# ansible-azure-vm-deploy

Ansible project for automating end-to-end Azure VM provisioning using `azure.azcollection`.  
Designed for **Ansible Automation Platform (AAP)** — all collections are bundled in the custom Execution Environment, no local collection installs required.

---

## What This Provisions

Starting from a blank Azure subscription:

1. **Resource Group**
2. **Virtual Network + Subnet**
3. **Network Security Group** (with configurable rules)
4. **Storage Account** (boot diagnostics)
5. **Public IP Address** (optional)
6. **Network Interface Card**
7. **Virtual Machine** (RHEL 9 by default, configurable)
8. **OS Baseline Configuration** (packages, firewall, hostname, MOTD)

---

## Repository Structure

```
├── execution-environment.yml     # ansible-builder spec for the custom EE
├── ansible.cfg                   # Minimal config — AAP manages inventory/creds
├── group_vars/all/vars.yml       # Non-sensitive defaults (region, sizes, naming)
├── playbooks/
│   ├── provision_infra.yml       # Job Template 1: RG, VNet, NSG, Storage
│   ├── deploy_vm.yml             # Job Template 2: PIP, NIC, VM
│   ├── configure_vm.yml          # Job Template 3: OS baseline config
│   └── destroy_infra.yml         # Job Template 4: Full teardown (gated)
└── roles/
    ├── azure_resource_group/
    ├── azure_network/
    ├── azure_security_group/
    ├── azure_storage/
    ├── azure_public_ip/
    ├── azure_vm/
    └── azure_vm_configure/
```

---

## AAP Setup Guide

### 1. Build and Register the Execution Environment

```bash
# Install ansible-builder if not already present
pip install ansible-builder

# Build the EE image
ansible-builder build \
  --file execution-environment.yml \
  --tag your-registry.example.com/azure-ee:latest \
  --container-runtime podman

# Push to your registry
podman push your-registry.example.com/azure-ee:latest
```

In AAP: **Administration → Execution Environments → Add**  
- Name: `Azure EE`  
- Image: `your-registry.example.com/azure-ee:latest`  
- Pull policy: `Always`

---

### 2. Configure the Azure Credential

In AAP: **Resources → Credentials → Add**  
- Credential Type: `Microsoft Azure Resource Manager`  
- Fields: Subscription ID, Client ID, Client Secret, Tenant ID

This injects `AZURE_SUBSCRIPTION_ID`, `AZURE_CLIENT_ID`, `AZURE_SECRET`, and `AZURE_TENANT` as environment variables at job runtime — no secrets in the repo.

---

### 3. Add the Project

In AAP: **Resources → Projects → Add**  
- Source Control Type: `Git`  
- Source Control URL: `https://github.com/mlowcher61/<this-repo>.git`   
- Branch/Tag/Commit: `main`  
- Update on launch: `enabled`

---

### 4. Configure the Dynamic Inventory Source

In AAP: **Resources → Inventories → Add → Add Inventory**  
- Name: `Azure Dynamic`

Then add an **Inventory Source**:  
- Source: `Microsoft Azure Resource Manager`  
- Credential: (the Azure credential from step 2)  
- Update on launch: `enabled`  
- Overwrite: `enabled`

This populates host groups based on Azure resource tags. The `configure_vm.yml` playbook targets the group `tag_managed_by_ansible_automation_platform`.

For `provision_infra.yml` and `deploy_vm.yml`, also create a **localhost inventory**:
- Add a host manually: `localhost` with variable `ansible_connection: local`

---

### 5. Create Job Templates

#### Job Template 1 — Provision Infrastructure

| Field | Value |
|---|---|
| Name | `Azure - Provision Infrastructure` |
| Inventory | `localhost inventory` |
| Project | `ansible-azure-vm-deploy` |
| Playbook | `playbooks/provision_infra.yml` |
| Execution Environment | `Azure EE` |
| Credentials | Azure Resource Manager credential |
| Extra vars / Survey | `env`, `project`, `az_location` |

#### Job Template 2 — Deploy VM

| Field | Value |
|---|---|
| Name | `Azure - Deploy VM` |
| Inventory | `localhost inventory` |
| Project | `ansible-azure-vm-deploy` |
| Playbook | `playbooks/deploy_vm.yml` |
| Execution Environment | `Azure EE` |
| Credentials | Azure RM credential + Machine/Custom credential for VM auth |

**Recommended Survey variables:**

| Variable | Type | Required | Notes |
|---|---|---|---|
| `env` | Text | Yes | dev / staging / prod |
| `project` | Text | Yes | Unique project name |
| `az_vm_size` | Text | No | Default: Standard_D2s_v3 |
| `az_vm_auth_type` | Multiple choice | Yes | password \| ssh |
| `az_admin_password` | Password | Conditional | Required if auth_type=password |
| `az_ssh_public_key` | Textarea | Conditional | Required if auth_type=ssh |
| `az_public_ip_enabled` | Multiple choice | No | true \| false |

#### Job Template 3 — Configure VM

| Field | Value |
|---|---|
| Name | `Azure - Configure VM` |
| Inventory | `Azure Dynamic` |
| Project | `ansible-azure-vm-deploy` |
| Playbook | `playbooks/configure_vm.yml` |
| Execution Environment | `Azure EE` |
| Credentials | Azure RM credential + Machine credential (SSH or password) |

#### Job Template 4 — Destroy Infrastructure

| Field | Value |
|---|---|
| Name | `Azure - Destroy Infrastructure` |
| Inventory | `localhost inventory` |
| Project | `ansible-azure-vm-deploy` |
| Playbook | `playbooks/destroy_infra.yml` |
| Execution Environment | `Azure EE` |
| Credentials | Azure RM credential |
| Extra vars on launch | `enabled` — user must pass `confirm_destroy=true` |

---

### 6. Create the Workflow Job Template

In AAP: **Resources → Templates → Add → Add Workflow Template**  
- Name: `Azure - Full VM Deployment`  
- Inventory: `localhost inventory` (overridden per node below)

**Workflow nodes:**

```
[Start]
   └── Provision Infrastructure  (on success)
         └── Deploy VM           (on success)
               └── Configure VM  (on success)
                     └── [End]
```

Pass `env` and `project` as workflow-level extra vars so all three Job Templates share the same values.

---

## Key Variables

All variables can be overridden via AAP Job Template extra vars or Survey.  
See `group_vars/all/vars.yml` for full defaults.

| Variable | Default | Description |
|---|---|---|
| `env` | `dev` | Environment name (used in resource naming) |
| `project` | `myproject` | Project name (used in resource naming) |
| `az_location` | `eastus` | Azure region |
| `az_vm_size` | `Standard_D2s_v3` | VM SKU |
| `az_os_publisher` | `RedHat` | OS image publisher |
| `az_os_offer` | `RHEL` | OS image offer |
| `az_os_sku` | `9-lvm-gen2` | OS image SKU |
| `az_public_ip_enabled` | `true` | Create public IP? |
| `az_vm_auth_type` | `password` | `password` or `ssh` |
| `az_vnet_address_prefix` | `10.10.0.0/16` | VNet CIDR |
| `az_subnet_prefix` | `10.10.1.0/24` | Subnet CIDR |

---

## Resource Naming Convention

All resources follow the pattern: `<project>-<environment>-<resource>`

Example with `project=webapp` and `env=prod`:
- Resource Group: `webapp-prod-rg`
- VNet: `webapp-prod-vnet`
- NSG: `webapp-prod-nsg`
- VM: `webapp-prod-vm01`
- Storage: `webappprodsa` *(alphanumeric only, 24 char max)*

---

## Tagging Strategy

All resources are tagged with:
```yaml
managed_by: ansible-automation-platform
environment: <env>
project: <project>
provisioned_date: <date>
```

The `configure_vm.yml` playbook uses the `managed_by` tag to target hosts via dynamic inventory group `tag_managed_by_ansible_automation_platform`.

---

## Adding More VMs

To deploy additional VMs in the same environment, override `az_vm_name` and `az_nic_name` via extra_vars on the Deploy VM Job Template. All shared infrastructure (VNet, NSG, etc.) is idempotent and will not be recreated.

---

## Destroy Safety Gate

The `destroy_infra.yml` playbook will fail immediately unless `confirm_destroy=true` is passed as an extra variable. Configure the AAP Job Template to prompt for extra vars on launch and **do not** set a default value for `confirm_destroy`.
