---
title: Azure Deployment
description: How to deploy the xianix-agent on an Azure Linux VM with no public IP and secrets in Key Vault.
---

This guide documents how the `xianix-agent` Docker image is deployed on an Azure Linux VM. The VM has **no public IP and no inbound ports open** — all internet traffic is outbound-only via a NAT Gateway. Secrets are stored in **Azure Key Vault** and retrieved at runtime via the VM's **system-assigned managed identity** — no plain-text `.env` file is kept on disk. The agent connects to the Docker socket and automatically pulls and manages `xianix-executor` containers.

---

## Provisioned Resources

| Resource | Value |
|---|---|
| Resource Group | `xianix-agent-rg` |
| Location | `norwayeast` |
| VM Name | `xianix-agent-vm` |
| VM Size | `Standard_B2s` |
| OS | Ubuntu 22.04 LTS |
| Public IP | None (no inbound access) |
| Admin User | `azureuser` |
| Key Vault | `xianix-kv-agent` |
| NAT Gateway | `xianix-agent-natgw` |
| Bastion | `xianix-agent-bastion` (Developer SKU — free) |
| NSG | `xianix-agent-vmNSG` (deny all inbound from Internet; allow port 22 from Azure platform) |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│  Azure VNet: xianix-agent-vmVNET                    │
│                                                     │
│  NSG: deny all inbound from Internet                │
│                                                     │
│  ┌──────────────────────────────────┐               │
│  │     Azure VM (xianix-agent-vm)   │               │
│  │                                  │               │
│  │  systemd: xianix-agent.service   │               │
│  │        │                         │               │
│  │        ▼                         │               │
│  │  /etc/xianix/start-agent.sh      │               │
│  │        │                         │               │
│  │        │  1. GET token (IMDS)    │               │
│  │        ▼                         │               │
│  │  Azure Key Vault ◄───────────────│── Managed Identity
│  │  xianix-kv-agent                 │               │
│  │        │                         │               │
│  │        │  2. Fetch secrets       │               │
│  │        ▼                         │               │
│  │  docker run xianix-agent ────────│── Docker socket
│  │        │                         │       │       │
│  │        ▼                         │       ▼       │
│  │  xianix-executor (per task)      │  auto-pulled  │
│  └──────────────────────────────────┘               │
│                    │ outbound only                   │
│                    ▼                                 │
│          NAT Gateway (outbound)                     │
└─────────────────────────────────┬───────────────────┘
                                  │
                                  ▼
                           Internet (Docker Hub,
                           Xians platform, APIs)
```

---

## Setup Steps

### 1. Create the Resource Group

```bash
az group create \
  --name xianix-agent-rg \
  --location norwayeast
```

### 2. Create the VM

```bash
az vm create \
  --resource-group xianix-agent-rg \
  --name xianix-agent-vm \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys
```

### 3. Install Docker

```bash
az vm run-command invoke \
  --resource-group xianix-agent-rg \
  --name xianix-agent-vm \
  --command-id RunShellScript \
  --scripts "
    curl -fsSL https://get.docker.com | sh
    usermod -aG docker azureuser
    systemctl enable docker
    systemctl start docker
  "
```

### 4. Enable System-Assigned Managed Identity

```bash
az vm identity assign \
  --resource-group xianix-agent-rg \
  --name xianix-agent-vm
```

This returns the identity's `principalId`, which is used in the next step.

### 5. Create the Key Vault

```bash
az keyvault create \
  --name xianix-kv-agent \
  --resource-group xianix-agent-rg \
  --location norwayeast \
  --enable-rbac-authorization false
```

### 6. Grant the VM Identity Access to Secrets

```bash
az keyvault set-policy \
  --name xianix-kv-agent \
  --resource-group xianix-agent-rg \
  --object-id <vm-principal-id> \
  --secret-permissions get list
```

### 7. Pre-pull Docker Images

```bash
az vm run-command invoke \
  --resource-group xianix-agent-rg \
  --name xianix-agent-vm \
  --command-id RunShellScript \
  --scripts "
    docker pull 99xio/xianix-agent:latest
    docker pull 99xio/xianix-executor:latest
  "
```

### 8. Create NAT Gateway for Outbound-Only Internet Access

```bash
az network public-ip create \
  --resource-group xianix-agent-rg \
  --name xianix-agent-nat-ip \
  --location norwayeast \
  --sku Standard \
  --allocation-method Static

az network nat gateway create \
  --resource-group xianix-agent-rg \
  --name xianix-agent-natgw \
  --location norwayeast \
  --public-ip-addresses xianix-agent-nat-ip \
  --idle-timeout 10

az network vnet subnet update \
  --resource-group xianix-agent-rg \
  --vnet-name xianix-agent-vmVNET \
  --name xianix-agent-vmSubnet \
  --nat-gateway xianix-agent-natgw
```

### 9. Remove the VM Public IP and Lock Down Inbound Traffic

```bash
az network nic ip-config update \
  --resource-group xianix-agent-rg \
  --nic-name xianix-agent-vmVMNic \
  --name ipconfigxianix-agent-vm \
  --remove publicIpAddress

az network nsg rule delete \
  --resource-group xianix-agent-rg \
  --nsg-name xianix-agent-vmNSG \
  --name default-allow-ssh

az network nsg rule create \
  --resource-group xianix-agent-rg \
  --nsg-name xianix-agent-vmNSG \
  --name deny-all-inbound \
  --priority 200 \
  --direction Inbound \
  --access Deny \
  --protocol '*' \
  --source-address-prefixes Internet \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges '*'

az network nsg rule create \
  --resource-group xianix-agent-rg \
  --nsg-name xianix-agent-vmNSG \
  --name allow-bastion-ssh \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 168.63.129.16 \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 22

az network public-ip delete \
  --resource-group xianix-agent-rg \
  --name xianix-agent-vmPublicIP
```

---

## Populating Secrets

### Via CLI

```bash
az keyvault secret set --vault-name xianix-kv-agent --name XIANS-SERVER-URL    --value "https://app.xians.ai"
az keyvault secret set --vault-name xianix-kv-agent --name XIANS-API-KEY        --value "<your-xians-api-key>"
az keyvault secret set --vault-name xianix-kv-agent --name ANTHROPIC-API-KEY    --value "<your-anthropic-api-key>"
az keyvault secret set --vault-name xianix-kv-agent --name GITHUB-TOKEN         --value "<your-github-token>"
az keyvault secret set --vault-name xianix-kv-agent --name AZURE-DEVOPS-TOKEN   --value "<your-ado-token>"
az keyvault secret set --vault-name xianix-kv-agent --name EXECUTOR-IMAGE       --value "99xio/xianix-executor:latest"
az keyvault secret set --vault-name xianix-kv-agent --name CONTAINER-MEMORY-MB  --value "1024"
az keyvault secret set --vault-name xianix-kv-agent --name CONTAINER-CPU-COUNT  --value "1"
```

### Via Azure Portal

Navigate to **Key Vault `xianix-kv-agent` → Secrets → Generate/Import** and add each secret listed above.

---

## How It Works at Runtime

The VM runs `/etc/xianix/start-agent.sh` as a systemd service on boot. The script:

1. Requests a short-lived bearer token from the **Azure Instance Metadata Service (IMDS)** using the VM's managed identity — no stored credentials needed.
2. Calls the Key Vault REST API to fetch each secret.
3. Passes the secrets as environment variables directly to `docker run`.

---

## Operations Reference

Since the VM has no public IP, all management is done via the Azure CLI. Define an optional alias to keep commands shorter:

```bash
alias vmrun='az vm run-command invoke \
  --resource-group xianix-agent-rg \
  --name xianix-agent-vm \
  --command-id RunShellScript \
  --scripts'
```

### Start the Agent

```bash
az vm run-command invoke \
  --resource-group xianix-agent-rg \
  --name xianix-agent-vm \
  --command-id RunShellScript \
  --scripts "sudo systemctl start xianix-agent"
```

### Check Status and Logs

```bash
# Service status
az vm run-command invoke ... --scripts "sudo systemctl status xianix-agent --no-pager"

# Agent container logs (last 50 lines)
az vm run-command invoke ... --scripts "docker logs --tail 50 xianix-agent 2>&1"

# List all running containers
az vm run-command invoke ... --scripts "docker ps"
```

### Tail Logs Continuously via Azure Bastion

`az vm run-command invoke` is one-shot and cannot stream. For live logs, connect via **Azure Bastion** (already provisioned, Developer SKU — free):

**Browser-based SSH (Developer SKU — no extra setup)**

Go to **Azure Portal → Virtual Machines → `xianix-agent-vm` → Connect → Bastion**, enter `azureuser` and your SSH private key, then run:

```bash
docker logs -f xianix-agent
```

**CLI-based SSH (Standard SKU — ~$0.19/hr)**

```bash
az network bastion update \
  --resource-group xianix-agent-rg \
  --name xianix-agent-bastion \
  --sku Standard \
  --enable-tunneling true

az network bastion ssh \
  --resource-group xianix-agent-rg \
  --name xianix-agent-bastion \
  --target-resource-id "$(az vm show -g xianix-agent-rg -n xianix-agent-vm --query id -o tsv)" \
  --auth-type ssh-key \
  --username azureuser \
  --ssh-key ~/.ssh/id_rsa
```

### Restart / Stop the Agent

```bash
az vm run-command invoke ... --scripts "sudo systemctl restart xianix-agent"
az vm run-command invoke ... --scripts "sudo systemctl stop xianix-agent"
```

### Rotate a Secret

```bash
az keyvault secret set \
  --vault-name xianix-kv-agent \
  --name ANTHROPIC-API-KEY \
  --value "<new-key>"

az vm run-command invoke ... --scripts "sudo systemctl restart xianix-agent"
```

### Update the Agent Image

```bash
az vm run-command invoke ... --scripts "docker pull 99xio/xianix-agent:latest && sudo systemctl restart xianix-agent"
```

### Update the Executor Image

The agent does **not** auto-pull the executor image — it always uses the locally cached version. To update:

```bash
az vm run-command invoke ... --scripts "docker pull 99xio/xianix-executor:latest"
```

### Start / Stop / Restart the VM

```bash
az vm start   --resource-group xianix-agent-rg --name xianix-agent-vm
az vm stop    --resource-group xianix-agent-rg --name xianix-agent-vm
az vm restart --resource-group xianix-agent-rg --name xianix-agent-vm
```

The agent starts automatically on boot via the systemd service.

---

## Notes

- The VM has no public IP and no open inbound ports. It is unreachable from the internet.
- All outbound traffic flows through the NAT Gateway.
- To manage the VM, use **Azure Bastion** or `az vm run-command invoke`.
- The executor image is **not** auto-pulled by the agent — pre-pull it explicitly after updates.
- No secrets are stored on disk. The `/etc/xianix/` directory holds only the startup script.
- The `--restart unless-stopped` Docker flag and `Restart=on-failure` systemd directive together ensure the agent survives container crashes and VM reboots.
