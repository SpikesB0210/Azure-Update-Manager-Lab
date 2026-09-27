# Azure-Update-Manager-Lab
# SOP: Azure Update Manager Lab

*Patch management and compliance reporting · Modular Terraform + Azure Update Manager + PowerShell*
## 🎬 Watch Me Build This Lab!

https://www.loom.com/share/0aa79e794acf4624816900c73085c035
| | |
|---|---|
| **VMs** | DC01 (Domain Controller) · WS01 (Member Server) · WS02 (Member Server) |
| **OS / Domain** | Windows Server 2022 Datacenter: Azure Edition · `aumlab.local` |
| **Relationship to other labs** | Fully standalone. Run it first, last, or by itself |
| **Creates** | 27 resources in `rg-aumlab` |
| **Deploy time** | ~8 minutes. DC01 promotion is the longest step |
| **Cost** | 3 × `Standard_D2als_v7` while running. Always run `terraform destroy` when finished |

---

## How This Lab Fits Into the Series

| Lab | What it deploys | Relationship |
|---|---|---|
| Lab 1: NTFS File Server | DC01, FS01, CLIENT01, VNet, NSG, Key Vault in RG-FileServerLab | Standalone |
| Lab 2: Azure RBAC | 3 role assignments on FS01 only | Depends on Lab 1 |
| AUM Lab: Azure Update Manager | DC01, WS01, WS02, VNet, Key Vault in rg-aumlab | Fully independent |

> **Remote state:** Reuse the existing `RG-TerraformState` storage account. This lab uses its own key, `aum-lab.tfstate`, so it never touches the other labs' state.

## What This Lab Covers and Why It Matters

Unpatched systems are one of the most common root causes of security incidents. At scale, the challenge is knowing which machines are missing which patches, enforcing a consistent schedule, and proving compliance to auditors. Azure Update Manager replaces WSUS with a cloud-native service that enrolls machines through policy, patches on a defined window, and keeps a per-machine compliance record.

### What You Will Learn

| Skill | Why it matters |
|---|---|
| Modular Terraform | Each module owns one concern. This is how production Terraform is organized |
| Policy-based enrollment | Define the rule once, and every current and future VM in scope is enrolled |
| Maintenance windows | A contract with the business: when patches install, which classifications, and reboot behavior |
| Assessment vs patching | Two separate operations. A VM can be assessed and never patched |
| On-demand assessment | What an ops team runs the moment a critical CVE is announced |
| Structured compliance export | Machine-readable evidence for SIEMs, ticketing systems, and audits |

## Architecture

<img width="860" height="540" alt="Image" src="https://github.com/user-attachments/assets/cd79fdba-41ad-4f7e-b472-f093ec55a6bf" />

## Why Each Component Exists

| Component | Purpose |
|---|---|
| `modules/networking` | VNet (10.0.0.0/16), subnet (10.0.1.0/24), and an NSG allowing RDP from your IP only. The subnet association is a separate resource. Without it, the NSG protects nothing |
| `modules/keyvault` | Stores the admin password. `enable_rbac_authorization = true` is required, or the deployer's role is ignored and secret writes return 403 |
| `modules/compute` | 3 VMs. DC01 has a static IP (10.0.1.4) because it is the DNS server. Every VM uses `patch_mode = "AutomaticByPlatform"` so Azure can drive patching |
| DC promotion extension | Installs AD DS and creates `aumlab.local`. Uses `-NoRebootOnCompletion` plus a delayed reboot so the extension reports success cleanly |
| Domain join extensions | Point DNS at DC01, wait for the domain to resolve, retry `Add-Computer` every 30s. `depends_on` makes them wait for promotion |
| `protected_settings` | Extension commands contain credentials, so they are encrypted instead of visible in the portal |
| Azure Policy (`59efceea`) | Enrolls every VM in the resource group in periodic assessment. Needs a managed identity and Virtual Machine Contributor because it modifies VMs |
| Maintenance Configuration | Every Sunday, 2:00 AM Eastern, 3 hours. Critical, Security, UpdateRollup. Reboot if required |
| Maintenance Assignments (×3) | Link the schedule to each VM. Without them, VMs are assessed but never patched |
| `scripts/validate-lab.ps1` | Reports PASS / FAIL / NO DATA per VM and exports `aum-compliance-report.json` |

## Prerequisites: Verify Before Starting

```powershell
terraform -version   # Must be >= 1.5.0
az version           # Any recent version
pwsh --version       # PowerShell 7+

az login
az account show      # Confirm the correct subscription

# Register the resource providers. Both must show: Registered
az provider register --namespace Microsoft.Maintenance
az provider register --namespace Microsoft.GuestConfiguration
az provider show --namespace Microsoft.Maintenance --query registrationState -o tsv
az provider show --namespace Microsoft.GuestConfiguration --query registrationState -o tsv

# Confirm the VM size is available to your subscription
az vm list-skus --location eastus --size Standard_D2als_v7 `
  --query "[].{Size:name, Restricted:join(',', restrictions[].reasonCode)}" -o table
# A blank Restricted column means allowed
```

> **Mac users:** set `pwsh` as the VS Code default terminal profile (Command Palette → *Terminal: Select Default Profile*) so shell integration works.

## Step 1: Clone the Repo

```powershell
git clone https://github.com/<your-username>/azure-update-manager-lab.git
cd azure-update-manager-lab
```

## Step 2: Review the Terraform Files

| File | Notes |
|---|---|
| [`backend.tf`](../backend.tf) | Set `storage_account_name` to your state storage account |
| [`versions.tf`](../versions.tf) | Terraform >= 1.5.0, azurerm ~> 3.100, random ~> 3.6 |
| [`variables.tf`](../variables.tf) | `admin_password` is `sensitive` and has no default |
| [`main.tf`](../main.tf) | Resource group plus the 4 module calls. Terraform infers the build order from references |
| [`modules/compute/main.tf`](../modules/compute/main.tf) | VM size, image, and patch settings |
| [`modules/update-manager/main.tf`](../modules/update-manager/main.tf) | **Set `start_date_time` to a future date** |

> **HCL rule:** a block written on one line can hold only one argument. Anything with two or more arguments must be on multiple lines, or `terraform init` fails.

## Step 3: Configure Variables

```powershell
# Create tfvars with your current public IP
$myIp = (Invoke-RestMethod -Uri "https://api.ipify.org").Trim()
Copy-Item terraform.tfvars.example terraform.tfvars
(Get-Content terraform.tfvars) -replace 'YOUR_PUBLIC_IP', $myIp | Set-Content terraform.tfvars
Get-Content terraform.tfvars          # Confirm allowed_rdp_ip ends in /32

# Admin password: masked input, never written to a file
$env:TF_VAR_admin_password = Read-Host "VM admin password" -MaskInput
[bool]$env:TF_VAR_admin_password      # Expect: True
```

> **Password rules:** at least 12 characters with upper, lower, number, and symbol. **No single quote (`'`)**, because the extension scripts wrap the password in single quotes.
> **Environment variables only last for the terminal session.** Set it again after reopening the terminal.

## Step 4: Initialize and Validate

```powershell
terraform init        # Expect: Successfully configured the backend "azurerm"
terraform fmt -recursive
terraform validate    # Expect: Success! The configuration is valid.
```

| Checkpoint | Expected |
|---|---|
| `init` | Modules found, azurerm 3.117.x and random 3.x installed, backend configured |
| `validate` | `Success! The configuration is valid.` |

## Step 5: Plan and Deploy

```powershell
terraform plan -out="aum-lab.tfplan"
```

Expected: **`Plan: 27 to add, 0 to change, 0 to destroy.`**

| Source | Resources |
|---|---|
| Root | 1 (resource group) |
| networking | 4 |
| keyvault | 4 (random suffix, vault, role assignment, secret) |
| compute | 12 (3 public IPs, 3 NICs, 3 VMs, 3 extensions) |
| update-manager | 6 (policy, role assignment, maintenance config, 3 assignments) |

```powershell
terraform apply "aum-lab.tfplan"
```

Observed timings:

| Resource | Time |
|---|---|
| VMs (parallel) | ~1 min |
| Maintenance assignments | ~20 sec each |
| **SetupDC** (AD DS + forest) | **4m19s** |
| JoinDomain (WS01 / WS02) | ~1.5–2 min, starting only after SetupDC |

> **If apply fails with `SkuNotAvailable`:** everything else stays in state. Change the size in `modules/compute/main.tf`, run `plan` again (the old plan file is stale), and apply. Terraform only builds what's missing.

## Step 6: Verify the Domain

"Extension succeeded" means the script ran, not that the join worked. Check from inside each VM with Run Command. Wait 2–3 minutes after apply for the post-join reboots.

```powershell
foreach ($vm in "DC01","WS01","WS02") {
    az vm run-command invoke -g rg-aumlab -n $vm --command-id RunPowerShellScript `
      --scripts "`$cs = Get-CimInstance Win32_ComputerSystem; '{0} | Domain: {1} | DomainRole: {2}' -f `$cs.Name, `$cs.Domain, `$cs.DomainRole" `
      --query "value[0].message" -o tsv
}
```

| VM | Expected | Meaning |
|---|---|---|
| DC01 | `aumlab.local`, DomainRole **5** | Primary Domain Controller |
| WS01 | `aumlab.local`, DomainRole **3** | Member server |
| WS02 | `aumlab.local`, DomainRole **3** | Member server |

`WORKGROUP` with role **2** means the join failed. See Troubleshooting.

## Step 7: Trigger an On-Demand Assessment

The policy assesses roughly every 24 hours. This forces it now, the same as clicking **Check for updates** in the portal.

```powershell
$subId = az account show --query id -o tsv
foreach ($vm in "DC01","WS01","WS02") {
    az rest --method POST `
      --url "https://management.azure.com/subscriptions/$subId/resourceGroups/rg-aumlab/providers/Microsoft.Compute/virtualMachines/$vm/assessPatches?api-version=2024-07-01"
    Write-Host "Assessment triggered: $vm" -ForegroundColor Cyan
}
```

No output from `az rest` is normal. Azure returns **202 Accepted** and runs the job in the background. Check progress:

```powershell
foreach ($vm in "DC01","WS01","WS02") {
    az vm get-instance-view -g rg-aumlab -n $vm `
      --query "{VM:name, Status:instanceView.patchStatus.availablePatchSummary.status, CritSec:instanceView.patchStatus.availablePatchSummary.criticalAndSecurityPatchCount}" -o table
}
```

Wait until all three show **Succeeded** (usually 5–10 minutes).

## Step 8: Validate Compliance

```powershell
./scripts/validate-lab.ps1
```

| Result | Meaning | Action |
|---|---|---|
| **PASS** | Assessment succeeded, 0 Critical/Security missing | None |
| **FAIL** | Assessment succeeded, Critical/Security patches missing | Patch (Step 9) |
| **NO DATA** | No completed assessment available | Wait, or re-run Step 7 |

The script uses the Azure CLI's existing sign-in and exports `aum-compliance-report.json` to the project root. A sample is in [`sample-compliance-report.json`](sample-compliance-report.json).

> **Why PASS on a new VM?** `2022-datacenter-azure-edition` is rebuilt monthly with the latest cumulative update, so new VMs start compliant. Older generic images usually show FAIL until patched.

## Step 9 (Optional): Emergency Patch Run

Proves the patching half of the workflow on one VM: assess → patch → re-assess.

```powershell
# What's missing
az vm assess-patches -g rg-aumlab -n WS01 `
  --query "availablePatches[].{Name:name, Classification:join(',', classifications)}" -o table

# Patch now (1-hour limit, reboot only if required)
az vm install-patches -g rg-aumlab -n WS01 `
  --maximum-duration PT1H --reboot-setting IfRequired `
  --classifications-to-include-win Critical Security UpdateRollup Definition Updates Tools `
  --query "{Status:status, Installed:installedPatchCount, Failed:failedPatchCount}" -o table

# Confirm
az vm assess-patches -g rg-aumlab -n WS01 --query "{CritSec:criticalAndSecurityPatchCount, Other:otherPatchCount}" -o table
./scripts/validate-lab.ps1
```

## Portal Verification Checklist

- [ ] **Azure Update Manager → Machines** (filter `rg-aumlab`): all 3 VMs assessed
- [ ] **Maintenance Configurations → aum-weekly-patches → Machines**: all 3 VMs linked
- [ ] **Policy → Assignments**: `aum-periodic-assessment` assigned to `rg-aumlab`

## Teardown

```powershell
[bool]$env:TF_VAR_admin_password    # destroy needs all required variables
terraform destroy                   # Review: 0 to add, 0 to change, 27 to destroy. Type yes

az group show --name rg-aumlab 2>&1                    # Expect: ResourceGroupNotFound
az keyvault list-deleted --query "[?starts_with(name,'kv-aum-')].name" -o tsv   # Expect: empty
```

### Subscription-wide cost check

```powershell
az group list --query "[].{Name:name, Location:location}" -o table
az vm list -d --query "[].{Name:name, RG:resourceGroup, Power:powerState}" -o table
az disk list --query "[].{Name:name, RG:resourceGroup, State:diskState}" -o table
az network public-ip list --query "[].{Name:name, RG:resourceGroup}" -o table
```

> Deallocated VMs stop compute billing, but **managed disks and static public IPs keep billing until deleted**. Set a budget alert in **Cost Management → Budgets** as a safety net.

## Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| `Unsupported argument` on `init` | Module missing a `variable` block for an input passed in `main.tf` | Declare the variable in that module's `main.tf` |
| `Invalid single-argument block definition` | Two or more arguments on one line | Put each argument on its own line |
| `SkuNotAvailable` | Size not offered to the subscription or region | `az vm list-skus`; choose an unrestricted x64 size |
| v6/v7 size fails to create | Gen1 image or SCSI controller | Use a Gen2 image and `disk_controller_type = "NVMe"` |
| Provider registration error | Provider not registered | Register and wait for `Registered` |
| 403 on Key Vault secret | RBAC mode off or propagation delay | Confirm `enable_rbac_authorization = true`; re-run apply |
| Policy assignment fails | Modify policy without identity | Add `identity { type = "SystemAssigned" }` and `location` |
| Maintenance config fails | `start_date_time` in the past | Set a future date |
| WS01/WS02 in `WORKGROUP` | DC not ready during join | Check DC01 → Extensions in the portal; re-run `terraform apply` |
| Validation shows NO DATA | Assessment not run or still running | Run Step 7, wait 5–10 minutes |
| `Connect-AzAccount` fails on macOS | WAM broker / Security Defaults block device code (`AADSTS530035`) | Not needed. `validate-lab.ps1` uses Azure CLI |
| Saved plan is stale | State changed after the plan was created | Run `terraform plan -out=...` again |
