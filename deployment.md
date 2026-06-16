# Azure Landing Zone — Bicep (AVM) + Azure DevOps — Deployment Runbook **v2**

**Goal:** Deploy the TRG platform landing zone with **Azure Verified Modules (Bicep)** via **Azure DevOps**, changing the **fewest files possible**. All values are collected once in a **discovery worksheet**, then dropped into a small set of files. Clear step-by-step.

> v2 supersedes v1. v1 documented two ADO approaches and full customization. v2 is the lean path: AVM + ADO accelerator, minimal edits, discovery-driven.

---

## 1. What gets deployed

The ALZ **IaC Accelerator (Bicep)** uses AVM modules from [`Azure/alz-bicep-accelerator`](https://github.com/Azure/alz-bicep-accelerator). It bootstraps your Azure DevOps repo + CI/CD, then deploys:

- **Management groups** under `TRG`: Platform (Connectivity, Identity subs) and Landing Zone (Application sub)
- **Azure Policy** (Microsoft ALZ baseline)
- **Logging** (Log Analytics in Management sub)
- **Hub network** (hub VNet + Azure Firewall in Connectivity sub)

Spoke VNets (Identity, App A, App B, Citrix) are added after, in one step (§7).

```
Tenant Root
└── TRG
    ├── Platform        → Connectivity sub (Hub VNet) · Identity sub (Identity spoke) · Management sub (logging)
    └── Landing Zone     → Application sub (App A + App B + Citrix spokes)
```

---

## 2. Discovery worksheet — gather these once

Fill every value before you start. These are the **only** inputs you need; everything else uses AVM/ALZ defaults.

### 2.1 Azure tenant & subscriptions
| # | Parameter | Your value | Notes |
|---|---|---|---|
| 1 | Tenant ID | | `az account show --query tenantId` |
| 2 | Tenant Root Group ID | | Usually = Tenant ID (GUID) |
| 3 | Management subscription ID | | Logging/automation |
| 4 | Connectivity subscription ID | | Hub network |
| 5 | Identity subscription ID | | Identity spoke |
| 6 | Application subscription ID | | App + Citrix spokes |

### 2.2 Regions & naming
| # | Parameter | Your value | Notes |
|---|---|---|---|
| 7 | Primary region | | e.g. `eastus` |
| 8 | Secondary region (or "none") | | Leave blank for single-region |
| 9 | Intermediate root MG id / name | `TRG` / `TRG` | Replaces default `alz` |
| 10 | Org naming prefix (optional) | | Applied to MG/resource names |

### 2.3 Networking (hub & spokes)
| # | Parameter | Your value | Notes |
|---|---|---|---|
| 11 | Hub VNet address space | e.g. `10.0.0.0/16` | Connectivity sub |
| 12 | Identity spoke address space | e.g. `10.1.0.0/24` | Identity sub |
| 13 | App A spoke address space | e.g. `10.2.0.0/24` | Application sub |
| 14 | App B spoke address space | e.g. `10.3.0.0/24` | Application sub |
| 15 | Citrix spoke address space | e.g. `10.4.0.0/23` | Application sub (sized for VDAs) |
| 16 | Azure Firewall SKU | `Standard` | Basic/Standard/Premium |
| 17 | Gateways needed? | VPN / ExpressRoute / none | |

### 2.4 Governance & DevOps
| # | Parameter | Your value | Notes |
|---|---|---|---|
| 18 | Security contact email | | Defender alerts |
| 19 | Ops/triage email | | Service health alerts |
| 20 | ADO organization URL | | `https://dev.azure.com/<org>` |
| 21 | ADO project name | | Created if missing |
| 22 | Agent type | Microsoft-hosted | Needs 1 parallel job (§4) |
| 23 | Module/bootstrap version to pin | latest stable release tag | Pin for reproducibility |

---

## 3. The only files you change

With the accelerator, edits are minimal. The interactive wizard writes the bootstrap file for you; you hand-edit just **one config file**, then **two parameter overrides**.

| Stage | File | What you set | Source |
|---|---|---|---|
| Bootstrap | `config/inputs.yaml` | Written by the wizard — **no manual edit** | Worksheet 1,3–6,20–23 |
| Bootstrap | `config/platform-landing-zone.yaml` | MG names (**TRG**), regions, `network_type: hubNetworking` | Worksheet 7–10 |
| After bootstrap | `templates/core/governance/mgmt-groups/int-root/main.bicepparam` | `managementGroupParentId` (tenant root), `emailSecurityContact`, triage email | Worksheet 2,18,19 |
| After bootstrap | `templates/networking/hubnetworking/main.bicepparam` | Hub `addressPrefixes`, firewall/gateway toggles | Worksheet 11,16,17 |

Subscription IDs and regions are **auto-injected** into all other `.bicepparam` files by the bootstrap — you don't touch them. Spoke address spaces are used in §7.

> That's it: **1 config file + 2 parameter files**. Everything else is default.

---

## 4. Step-by-step

### Step 1 — Prerequisites (once)
1. Create the 4 subscriptions; record IDs (worksheet 3–6).
2. Grant the engineer running this: **`Owner`** on `TRG` MG and each subscription, plus **`User Access Administrator` at root `/`** (Bicep requirement).
3. In Azure DevOps: enable billing and ensure **≥1 parallel job**; create a short-lived **PAT** (`token-1`) with scopes: Agent Pools (R&M), Build (R&E), Code (Full), Environment (R&M), Graph (R&M), Pipeline Resources (U&M), Project & Team (RW&M), Service Connections (RQ&M), Variable Groups (RC&M).
4. Install tooling locally: PowerShell 7+, Azure CLI, `az bicep install`, Git, VS Code.

### Step 2 — Authenticate
```pwsh
az login
az account set --subscription "<management-subscription-id>"
az account show
```

### Step 3 — Run the accelerator (writes inputs.yaml for you)
```pwsh
$m = Get-InstalledPSResource -Name ALZ 2>$null
if (-not $m) { Install-PSResource -Name ALZ } else { Update-PSResource -Name ALZ }
Deploy-Accelerator
```
Answer the wizard from your worksheet: IaC = **Bicep**, VCS = **Azure DevOps**, org/project, `token-1`, the 4 subscription IDs, parent MG = `TRG`, `networkType` = **hubNetworking**, pin the version (worksheet 23).

### Step 4 — Edit the one config file
When prompted, open `config/platform-landing-zone.yaml` and set:
```yaml
starter_locations: ["<primary-region>"]          # add secondary only if multi-region
network_type: "hubNetworking"
management_group_int_root_id:   "TRG"
management_group_int_root_name: "TRG"
# keep platform / connectivity / identity / landingzones; trim sandbox/decommissioned if unwanted
```
Save, then type `yes` to let the bootstrap build the ADO project, repos, pipelines, service connection (OIDC), and identity.

### Step 5 — Clone the generated repo and make the 2 overrides
```pwsh
git clone https://dev.azure.com/<org>/<project>/_git/<module-repo>
cd <module-repo>
```
Edit **`templates/core/governance/mgmt-groups/int-root/main.bicepparam`**:
```bicep
intRootConfig: {
  managementGroupParentId: '<tenant-root-group-guid>'   // worksheet 2
  ...
}
// in parPolicyAssignmentParameterOverrides:
emailSecurityContact: { value: '<security-email>' }     // worksheet 18
actionGroupEmail:      ['<triage-email>']               // worksheet 19
```
Edit **`templates/networking/hubnetworking/main.bicepparam`**:
```bicep
param hubNetworks = [
  {
    name: 'vnet-hub-trg-<region>'
    addressPrefixes: ['<hub-cidr>']                      // worksheet 11
    azureFirewallSettings: { deployAzureFirewall: true, azureSkuTier: '<sku>' }   // worksheet 16
    vpnGatewaySettings:    { deployVpnGateway: <true|false> }                     // worksheet 17
  }
]
```
> Single region: keep one region in every `parLocations`, remove the second hub object, set `deployPeering: false`.

### Step 6 — Deploy (this is the "run")
```
branch → commit → push → open PR → CI runs build + what-if → review → merge to main → CD deploys (approve the Apply gate)
```
There is **no single file to execute** — **merging to `main` triggers the CD pipeline**, which deploys each module's `main.bicep` in order: management groups/policy → logging → hub network.

### Step 7 — Add the spoke VNets (one file)
Use the AVM **subscription-vending / spoke** module ([`avm/ptn/lz/sub-vending`](https://github.com/Azure/bicep-registry-modules/tree/main/avm/ptn/lz/sub-vending)) to create each spoke VNet and peer it to the hub. Define all four in a single `spokes.bicepparam` (one entry per spoke) and add a final stage to the CD pipeline:

| Spoke | Subscription | MG | Address space | Peer to hub |
|---|---|---|---|---|
| Identity | Identity | Platform | worksheet 12 | Yes |
| App A | Application | Landing Zone | worksheet 13 | Yes |
| App B | Application | Landing Zone | worksheet 14 | Yes |
| Citrix | Application | Landing Zone | worksheet 15 | Yes |

> Citrix: route egress through the hub firewall (UDR → AzFW) and allow Citrix Cloud FQDNs/ports in firewall policy.

### Step 8 — Validate
- `TRG` MG exists; Platform & Landing Zone are children; subs in the right MGs
- Policy assigned; Log Analytics in Management sub
- Hub VNet + Firewall in Connectivity sub
- 4 spokes deployed and peered (`Connected`)
- Revoke `token-1`

---

## 5. Ongoing updates (Azure DevOps)

All changes follow one loop: **branch → edit `.bicepparam` → PR (CI what-if) → approve → merge (CD deploys)**. Keep branch policy on `main` (PR + green CI + reviewer). To upgrade AVM/ALZ versions, bump the pinned version on a branch, review the what-if, merge. Pin versions for production.

---

## 6. References
- IaC Accelerator: https://azure.github.io/Azure-Landing-Zones/accelerator/
- Bicep getting started / customization: https://azure.github.io/Azure-Landing-Zones/bicep/gettingstarted/
- Bootstrap (Phase 2): https://azure.github.io/Azure-Landing-Zones/accelerator/2_bootstrap/
- ADO prerequisites: https://azure.github.io/Azure-Landing-Zones/accelerator/1_prerequisites/azuredevops/
- Platform subs & permissions: https://azure.github.io/Azure-Landing-Zones/accelerator/1_prerequisites/platform-subscriptions/
- Repo: https://github.com/Azure/alz-bicep-accelerator
- Subscription vending (AVM): https://github.com/Azure/bicep-registry-modules/tree/main/avm/ptn/lz/sub-vending

---
*v2 — lean AVM + ADO path. Edit 1 config file + 2 parameter files; deploy by merging to `main`.*
