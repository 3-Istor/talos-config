# Talos Linux Configuration & Deployment Guide

Welcome to the 3-Istor Cloud Talos configuration repository. This repository uses a **GitOps & Patch-driven** approach to manage our Kubernetes OS (Talos Linux) securely.

## 🧠 Architecture Overview

Talos Linux configurations normally contain highly sensitive certificates and tokens. To prevent secret leaks and make collaboration easy, we split the configuration into two parts:

1. **Secrets (HashiCorp Vault)**: Core cryptographic materials are stored securely in our Vault.
2. **Configuration Patches (This Repository)**: Standard infrastructure configurations (DNS, NTP, Node Labels, extra arguments) are stored here as YAML patches.

When we run Terraform, the `siderolabs/talos` provider dynamically fetches the secrets from Vault, fetches the patches from this repository, merges them natively, and injects the final secure configuration into our OpenStack VMs.

## 🛠 Prerequisites

Before starting, ensure you have the following tools installed on your local machine:
- [Terraform](https://developer.hashicorp.com/terraform/downloads) (>= 1.5)
- [Vault CLI](https://developer.hashicorp.com/vault/downloads)
- [talosctl](https://www.talos.dev/docs/getting-started/talosctl/) (For interacting with the cluster later)

## 🔐 Step 1: Authenticate to Vault

Terraform needs to pull secrets from our Vault instance to assemble the Talos machine configuration. You must authenticate to Vault first.

Export the Vault address and login using your 3-Istor credentials:

```bash
export VAULT_ADDR="https://vault.3istor.com/"

# Depending on our specific auth method for the '3istor' realm (e.g., OIDC, userpass, LDAP):
vault login -method=oidc 
# OR if using a specific namespace/realm parameter depending on your setup:
# vault login -namespace=3istor -method=userpass username=<your_username>
```

Verify your token is active by reading the cluster secrets (Note: You only need read access):
```bash
vault kv get -format=json kvv2/talos/demo-cluster | jq -r '.data.data.machine_secrets' > secrets.yaml
```

## 📝 Step 2: Modifying the Configuration

We do not write the entire Talos configuration by hand. We only write **Strategic Merge Patches**.

If you need to change a system setting (e.g., adding a registry mirror, changing DNS, adding a kernel argument), simply edit the relevant `*-patch.yaml` file in this repository.

### Example: Adding a new Node Label
Edit `demo-patch.yaml`:

```yaml
machine:
  nodeLabels:
    team: 3-istor
    environment: demo
    new-label: my-feature   # <--- Add your changes here
```

Once modified, **commit and push** your changes to the `main` branch. Terraform will automatically pull the raw URL of this file during the next apply.

*(For a full list of available configuration fields, refer to the [Talos Machine Configuration Reference](https://www.talos.dev/docs/reference/configuration/).)*

## 🚀 Step 3: Deploying the Changes

Once your patch is pushed to this repository, navigate to the Terraform OpenStack directory on your local machine to apply the changes.

```bash
cd openstack/projects/3-istor-cloud/

# Ensure your OpenStack credentials are sourced
source admin-openrc.sh

# Apply the infrastructure
terraform init
terraform apply
```

Terraform will detect that the remote `demo-patch.yaml` file has changed, re-assemble the machine configuration, and apply it.

*(Note: Updating user-data on an existing OpenStack VM usually requires replacing the VM. If you are live-updating a running Talos node without recreating the VM, you will use `talosctl apply-config` instead, but Terraform remains the source of truth for initial provisioning).*
