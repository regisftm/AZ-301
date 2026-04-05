# Lab 1: Infrastructure Setup and FortiWeb HA Deployment

## Lab Overview

**Duration:** 45 minutes  
**Difficulty:** Intermediate  
**Prerequisites:** Completed AZ-201 or equivalent Azure networking experience, FortiFlex tokens available, Azure service principal credentials ready

### Objective

Deploy the foundational infrastructure for Redwood Industries' web application protection environment. You will create a dedicated resource group and virtual network, configure Azure Bastion for secure management access, and deploy a FortiWeb Active-Active HA cluster using an Azure ARM template into the existing network.

### What You'll Build

By the end of this lab, you will have:

- ✅ A dedicated resource group for all web protection resources
- ✅ A hub virtual network with four subnets (Bastion, External, Internal, Protected)
- ✅ Azure Bastion for secure, browser-based access to private VMs
- ✅ FortiWeb Active-Active HA cluster deployed via ARM template
- ✅ External Load Balancer distributing traffic across both FortiWeb nodes
- ✅ Initial access to the FortiWeb management interface verified

### Architecture

After Lab 1:

```text
Internet
    │
    ▼
External Load Balancer (Public IP)
    │
    ├──────────────────────────┐
    ▼                          ▼
FortiWeb Node 1 (fwb-vm0)   FortiWeb Node 2 (fwb-vm1)
   port1: 10.0.1.x             port1: 10.0.1.x
   port2: 10.0.2.x             port2: 10.0.2.x
    │                          │
    └──────────┬───────────────┘
               ▼
         Internal Subnet (10.0.2.0/24)
               │
               ▼
         Protected Subnet (10.0.3.0/24)
         [App Servers — deployed in Lab 3]
```

### Business Context

**Redwood Industries' Challenge:**

Redwood Industries is launching their first customer-facing web application on Azure. Their security team has mandated web application firewall (WAF) protection before the application goes live. A single WAF appliance creates an unacceptable single point of failure — their SLA requires 99.9% availability for this customer-facing service.

**The Solution:**

A FortiWeb Active-Active cluster with an Azure External Load Balancer provides:

- **Redundancy:** Two FortiWeb nodes share the traffic load — if one fails, the other continues without interruption
- **Scale:** Active-Active distributes inspection load across both nodes
- **Azure-native resilience:** Load balancer health probes detect failures automatically
- **WAF capabilities:** OWASP Top 10 protection, signature-based and behaviour-based threat detection

---

## Understanding FortiWeb Active-Active HA in Azure

### How Active-Active HA Works

Unlike Active-Passive HA, FortiWeb Active-Active mode means **both nodes handle production traffic simultaneously**. Configuration is kept in sync between nodes using FortiWeb's built-in configuration replication — covered in Lab 2.

```text
Traffic Flow (Active-Active):

Client Request
    │
    ▼
Azure External Load Balancer
    ├── Session A → FortiWeb Node 1 (inspects, forwards to app server)
    ├── Session B → FortiWeb Node 2 (inspects, forwards to app server)
    └── Session C → FortiWeb Node 1 (load balanced)

If Node 1 fails:
    ├── Existing sessions on Node 1: dropped (no session sync in AA mode)
    └── All new sessions → Node 2 (health probe removes Node 1 from rotation)
```

> [!NOTE]
> Active-Active FortiWeb in this workshop will not synchronise session state between nodes. Clients may need to re-establish connections during a node failure. For most HTTP/HTTPS applications, this is transparent to end users as browsers retry automatically.

### Why an ARM Template?

The FortiWeb HA deployment involves multiple interdependent Azure resources: two VMs, network interfaces, availability set (or availability zone), load balancer, backend pools, health probes, and load balancing rules. Deploying these manually would take over an hour and introduce configuration errors. The ARM template creates all resources consistently in minutes, and is the Fortinet-recommended deployment method for production HA clusters.

### The Three-Subnet Design

FortiWeb uses three functional subnets in this architecture:

| Subnet | CIDR | Purpose |
| --- | --- | --- |
| `external` | 10.0.1.0/24 | FortiWeb port1 — internet-facing, management access |
| `internal` | 10.0.2.0/24 | FortiWeb port2 — traffic to/from protected servers |
| `protected` | 10.0.3.0/24 | Application servers (no direct internet access) |

> [!NOTE]
> The `AzureBastionSubnet` (10.0.0.0/24) is a required fourth subnet. Azure Bastion uses it to provide browser-based SSH/RDP access to VMs in the protected subnet — without requiring public IPs on those VMs.

---

## Part A: Create the Resource Group

### Step 1: Create the Resource Group

#### 1.1 Navigate to Resource Groups

1. Log in to the Azure Portal at `https://portal.azure.com`
2. In the top search bar, type `resource groups`
3. Click **Resource groups** in the results

#### 1.2 Create New Resource Group

1. Click **+ Create** at the top of the page
2. Configure the following:

   | Setting | Value |
   | --- | --- |
   | **Subscription** | Your Azure subscription |
   | **Resource group** | `redwood-app-protection-rg` |
   | **Region** | `Canada Central` |

3. Click **Review + create**
4. Click **Create**

> [!NOTE]
> All AZ-301 resources are deployed into `redwood-app-protection-rg` and the `Canada Central` region. Keeping all resources in one region and one resource group simplifies cleanup at the end of the workshop.

### Validation

- ✅ Resource group `redwood-app-protection-rg` created in Canada Central

---

## Part B: Create the Virtual Network and Subnets

### Step 2: Create the Virtual Network

#### 2.1 Navigate to Virtual Network Creation

1. Navigate to your resource group: **redwood-app-protection-rg**
2. Click **+ Create**
3. In the Marketplace search field, type `virtual network`
4. Click **Virtual Network** (Microsoft service)
5. Click **Create**

#### 2.2 Configure Basic Settings

1. In the **Basics** tab, configure:

   | Setting | Value |
   | --- | --- |
   | **Subscription** | Your Azure subscription |
   | **Resource group** | `redwood-app-protection-rg` |
   | **Name** | `vnet-app-protection` |
   | **Region** | `Canada Central` |

2. Click **Next**

### Step 3: Configure Azure Bastion

#### 3.1 Enable Azure Bastion

1. In the **Security** tab:
   - Select the  **Enable Azure Bastion** box

2. Configure Bastion settings:

   | Setting | Value |
   | --- | --- |
   | **Bastion name** | `bas-app-protection` |
   | **Public IP address** | Click **Create a public IP address** |
   | **Public IP name** | `pip-bas-app-protection` |
   | **SKU** | Standard (default) |

3. Click **OK** to confirm the public IP
4. Click **Next**

> [!NOTE]
> Azure Bastion deployment typically takes 5–10 minutes after VNet creation. This is expected behaviour for Bastion services.

### Step 4: Configure IP Address Space and Subnets

#### 4.1 Set the VNet Address Space

1. In the **IP Addresses** tab:
   - **IPv4 address space:** `10.0.0.0/16`
  
2. In the **Subnets** space:
   - **Delete** the `default` subnet `10.0.0.0/24`
  
#### 4.2 Verify the Bastion Subnet

The `AzureBastionSubnet` should have been created automatically when you enabled Bastion:

- Verify the name is exactly `AzureBastionSubnet` (Azure requires this exact name)
- Set the address range to `10.0.0.0/26` clicking on the edit pencil in front of it.

> [!WARNING]
> The Bastion subnet name must be exactly `AzureBastionSubnet` — Azure will reject any other name. Do not rename it.

#### 4.3 Create the External Subnet

1. Click **+ Add subnet**
2. Configure:

   | Setting | Value |
   | --- | --- |
   | **Subnet name** | `external` |
   | **Starting address** | `10.0.1.0` |
   | **Size** | `/24` (256 addresses) |

3. Click **Add**

#### 4.4 Create the Internal Subnet

1. Click **+ Add subnet**
2. Configure:

   | Setting | Value |
   | --- | --- |
   | **Subnet name** | `internal` |
   | **Starting address** | `10.0.2.0` |
   | **Size** | `/24` (256 addresses) |

3. Click **Add**

#### 4.5 Create the Protected Subnet

1. Click **+ Add subnet**
2. Configure:

   | Setting | Value |
   | --- | --- |
   | **Subnet name** | `protected` |
   | **Starting address** | `10.0.3.0` |
   | **Size** | `/24` (256 addresses) |

3. Click **Add**

### Step 5: Review and Create the VNet

#### 5.1 Verify All Subnets

1. Click **Review + create**
2. Verify all four subnets are listed:

   | Subnet | Address Range |
   | --- | --- |
   | `AzureBastionSubnet` | 10.0.0.0/26 |
   | `external` | 10.0.1.0/24 |
   | `internal` | 10.0.2.0/24 |
   | `protected` | 10.0.3.0/24 |

3. Click **Create**

> [!NOTE]
> Wait for the VNet deployment to complete (1–2 minutes) before proceeding. The Bastion resource itself continues deploying in the background — you do not need to wait for it before starting Part C.

### Validation

- ✅ VNet `vnet-app-protection` created in Canada Central
- ✅ `AzureBastionSubnet` 10.0.0.0/24 present
- ✅ `external` subnet 10.0.1.0/24 present
- ✅ `internal` subnet 10.0.2.0/24 present
- ✅ `protected` subnet 10.0.3.0/24 present

---

## Part C: Prepare for ARM Template Deployment

This template is self-contained. Unlike traditional FortiWeb HA deployments, it does not require an Azure App Registration or service principal — HA operations are handled directly by the template at deploy time. The only thing you need before deploying is your two FortiFlex tokens.

### Step 6: Locate Your FortiFlex Tokens

Each FortiWeb node requires its own FortiFlex token. The tokens are applied directly as ARM template parameters and activate licensing automatically during the first boot sequence.

#### 6.1 Retrieve Your Tokens

Locate the FortiFlex Tokens in your welcome e-mail:

| Token | Description |
| --- | --- |
| **FortiFlex Token A** | Applied to FortiWeb Node 1 (`fweb-ha-vm1`) |
| **FortiFlex Token B** | Applied to FortiWeb Node 2 (`fweb-ha-vm2`) |

> [!TIP]
> Open a text editor (Notepad, VS Code) and paste both tokens now. You will enter them in Step 8 — they are long strings and easy to mistype.  

> [!NOTE]
> **How FortiFlex activation works in this template:** The tokens are injected into each VM's `customData` at deployment time. When FortiWeb boots for the first time, it reads the token from `customData` and contacts Fortinet's licensing servers automatically. No manual activation step is required in the GUI.

### Validation

- ✅ FortiFlex Token A copied and saved
- ✅ FortiFlex Token B copied and saved

---

## Part D: Deploy the FortiWeb HA Cluster

### Step 7: Create the ARM Template Spec

#### 7.1 Navigate to Template Specs

1. Navigate to your resource group: `redwood-app-protection-rg`
2. Click **+ Create**
3. In the **Search the Marketplace** field, type `template specs`
4. In the **Template specs** in the results
5. Click **+ Create > Template Specs**

#### 7.2 Configure the Template Spec

1. In the **Basics** tab, configure:

| Setting | Value |
|---|---|
| **Subscription** | Your Azure subscription |
| **Resource group** | `redwood-app-protection-rg` |
| **Name** | `fortiweb-ha-template` |
| **Location** | `Canada Central` |
| **Version** | `v1.0.0` |

2. Click **Next: Edit Template**

#### 7.3 Paste the ARM Template Content

1. In the template editor, select all existing content and delete it
2. Open the following URL in a new browser tab:
```
https://raw.githubusercontent.com/regisftm/AZ-301/refs/heads/main/arm-template/deploy_fwb_ha.json
```

3. Select all content (`Ctrl+A`) and copy it (`Ctrl+C`)
4. Return to the Azure Portal template editor and paste (`Ctrl+V`)
5. Click **Review + create**
6. Click **Create**

> [!NOTE]
> The template spec is a reusable container for the ARM template. Creating it first means
> you can redeploy the same template in future workshops without re-pasting the JSON.

### Step 8: Deploy the FortiWeb HA Cluster

#### 8.1 Open the Template for Deployment

1. After creation, click **fortiweb-ha-template** in the template spec list
2. Click **Deploy**

#### 8.2 Configure Basics

| Setting | Value |
|---|---|
| **Subscription** | Your Azure subscription |
| **Resource group** | `redwood-app-protection-rg` |
| **Region** | `Canada Central` |

#### 8.3 Configure Template Parameters

**Resource Naming:**

| Parameter | Value |
|---|---|
| **Resource Name Prefix** | `fweb-ha` |

> [!NOTE]
> The prefix `fweb-ha` is used to name all resources created by this template. Your
> FortiWeb VMs will be named `fweb-ha-vm1` and `fweb-ha-vm2`, the load balancer
> `fweb-ha-loadbalance`, and so on. Keep this prefix — it is referenced throughout
> the remaining labs.

**VM Configuration:**

| Parameter | Value |
|---|---|
| **Vm Sku** | `Standard_F2s_v2` |
| **Availability Options** | `Availability Set` |
| **Accelerated Networking** | `false` |
| **Vm Admin Username** | `fortiuser` |
| **Vm Admin Password** | Create a strong password — save it securely |
| **Vm Image Version** | `latest` |

> [!WARNING]
> The admin username cannot be `admin` or `root` — Azure will reject these. Use
> `fortiuser` as shown above.

> [!WARNING]
> The password must satisfy at least 3 of these 4 conditions: uppercase characters,
> lowercase characters, a digit, a special character. Save this password — you will
> need it to access both FortiWeb nodes throughout the workshop.

**Network Configuration:**

| Parameter | Value |
|---|---|
| **Vnet New Or Existing** | `existing` |
| **Vnet Resource Group** | `redwood-app-protection-rg` |
| **Vnet Name** | `vnet-app-protection` |
| **Vnet Address Prefix** | `10.0.0.0/16` |
| **Vnet Subnet1 Name** | `external` |
| **Vnet Subnet1 Prefix** | `10.0.1.0/24` |
| **Vnet Subnet2 Name** | `internal` |
| **Vnet Subnet2 Prefix** | `10.0.2.0/24` |

**Load Balancer:**

| Parameter | Value |
|---|---|
| **Load Balancer Type** | `Public` |

**FortiFlex Licensing:**

| Parameter | Value |
|---|---|
| **Fortiweb License Forti Flex A** | Your FortiFlex Token A (from Step 6.1) |
| **Fortiweb License Forti Flex B** | Your FortiFlex Token B (from Step 6.1) |

> [!NOTE]
> The tokens are injected into each VM's `customData` at deployment time. When each
> FortiWeb node boots for the first time, it reads the token and contacts Fortinet's
> licensing servers automatically — no manual activation in the GUI is required.

#### 8.4 Deploy

1. Click **Review + create**
2. Review all parameters — pay particular attention to the VNet name and subnet names,
   which must match exactly what you created in Part B
3. Click **Create**

> [!NOTE]
> The deployment typically takes **8–12 minutes**. The template is simultaneously
> creating: an availability set, two FortiWeb VMs, four network interfaces, three public
> IP addresses, a load balancer with backend pool, health probes, and load balancing
> rules. ☕

#### 8.5 Monitor Deployment Progress

1. Click the notification bell 🔔 at the top right to see deployment status
2. For detailed progress, navigate to `redwood-app-protection-rg > Deployments`

### Validation

When deployment completes, verify the following resources exist in
`redwood-app-protection-rg`:

- [ ] `fweb-ha-vm1` — FortiWeb Node 1 (VM)
- [ ] `fweb-ha-vm2` — FortiWeb Node 2 (VM)
- [ ] `fweb-ha-availabilitySet` — Availability set containing both VMs
- [ ] `fweb-ha-loadbalance` — External Load Balancer
- [ ] `fweb-ha-loadbalance-IP` — Load Balancer public IP
- [ ] `fweb-ha-nicPublic-IP1` and `fweb-ha-nicPublic-IP2` — VM management public IPs
- [ ] `fweb-ha-securityGroup` — Network Security Group

---

## Part E: Access FortiWeb and Verify the Cluster

### Step 9: Locate the FortiWeb Management IPs

Each FortiWeb node has its own public IP for management access on port1 (the `external`
subnet interface).

#### 9.1 Find the Public IPs

1. Navigate to `redwood-app-protection-rg`
2. Locate the following public IP resources and note their assigned IP addresses:

| Resource | Purpose |
|---|---|
| `fweb-ha-nicPublic-IP1` | Management access — FortiWeb Node 1 (`fweb-ha-vm1`) |
| `fweb-ha-nicPublic-IP2` | Management access — FortiWeb Node 2 (`fweb-ha-vm2`) |
| `fweb-ha-loadbalance-IP` | Application traffic — shared VIP for both nodes |

> [!TIP]
> Open your text editor and record all three IP addresses now. You will use the
> management IPs in Steps 10–13, and the load balancer IP in Lab 4 when you
> configure and test traffic steering.

### Step 10: Connect to the FortiWeb GUI

#### 10.1 Open the FortiWeb Management Interface — Node 1

1. Open a new browser tab and navigate to:
```
https://<fweb-ha-nicPublic-IP1>:8443
```

2. You will see a certificate warning — this is expected (self-signed certificate)
3. Click **Advanced** → **Proceed** (or the equivalent in your browser)

#### 10.2 Log In

| Field | Value |
|---|---|
| **Username** | `fortiuser` |
| **Password** | The VM admin password you set in Step 8.3 |

> [!NOTE]
> `fortiuser` is the VM operating system administrator account, provisioned by Azure
> during deployment. You will configure the FortiWeb `admin` account password separately
> in Step 11.

#### 10.3 Verify Licensing Status

After login, navigate to **System > Status > Status** and confirm:

- The **Licence** section shows a valid FortiFlex licence
- The VM serial number is displayed

> [!NOTE]
> If the licence status shows as **Pending** or **Unlicensed**, wait 2–3 minutes and
> refresh the page. The node may still be completing its first-boot activation sequence.
> The FortiWeb VM must have internet connectivity to reach Fortinet's licensing servers —
> if activation does not complete within 5 minutes, check that the NSG allows outbound
> traffic.

### Step 11: Configure the FortiWeb Admin Account

The FortiWeb `admin` account is the primary management account for all GUI and CLI
configuration tasks. On Azure deployments, this account has no password set by default
and must be configured before use.

#### 11.1 Open the CLI Console

1. In the FortiWeb GUI, click the **CLI Console** icon in the top-right toolbar

#### 11.2 Set the Admin Password on Node 1
```
config system admin
    edit "admin"
        set password "YourStrongPassword1!"
    next
end
```

> [!WARNING]
> Replace `YourStrongPassword1!` with a strong password of your choice. The `admin`
> account password **must not be empty** on Azure-deployed FortiWeb instances. You will
> use the `admin` account for all configuration tasks in Labs 2 through 5 — save it now.

#### 11.3 Set the Admin Password on Node 2

1. Open a new browser tab and navigate to:
```
https://<fweb-ha-nicPublic-IP2>:8443
```

2. Log in with `fortiuser` credentials
3. Repeat Steps 11.1 and 11.2 using the **same** `admin` password

> [!NOTE]
> Use the same `admin` password on both nodes. In Lab 2 you will configure
> configuration replication from Node 1 to Node 2 — consistent credentials simplify
> that process.

### Step 12: Verify HA Cluster Status

#### 12.1 Check HA Settings — Node 1

1. In the FortiWeb GUI on Node 1, navigate to **System > High Availability > Settings**
2. Verify the following values, which were injected automatically by the ARM template:

| Setting | Expected Value |
|---|---|
| **Mode** | `Active-Active High Volume` |
| **Group Name** | `az-301-ha-group` |
| **Group ID** | `5` |
| **Override** | `Disabled` |

> [!NOTE]
> These settings were configured automatically via the `customData` parameter during VM
> provisioning. You do not need to modify them — this is the expected Active-Active
> configuration for this workshop.

#### 12.2 Check HA Settings — Node 2

1. Switch to the browser tab for Node 2
2. Navigate to **System > High Availability > Settings**
3. Verify the same values as Node 1 — both nodes must have identical HA group settings
   for the cluster to form correctly

### Step 13: Verify the Azure Load Balancer

#### 13.1 Check the Backend Pool

1. In the Azure Portal, navigate to `redwood-app-protection-rg`
2. Click on **fweb-ha-loadbalance**
3. Click **Settings > Backend pools**
4. Click on **FwbHaLBBackendAddrPool**
5. Verify both `fweb-ha-vm1` and `fweb-ha-vm2` are listed as backend pool members

#### 13.2 Check Health Probe Status

1. Click **Monitoring > Insights** or navigate to **Settings > Load balancing rules**
2. Verify both nodes are responding to health probes on ports 80 and 443

> [!NOTE]
> If a node shows as **Unhealthy**, verify the VM is in **Running** state and has
> completed its boot sequence. Health probe failures during initial deployment typically
> resolve within 2–3 minutes. The NSG created by the template already allows inbound
> traffic on ports 80, 443, 8080, and 8443 — no additional NSG rules are needed.

### Validation

- [ ] FortiWeb GUI accessible on both nodes at port 8443
- [ ] FortiFlex licences show as valid on both nodes
- [ ] `admin` account password configured on both nodes
- [ ] HA settings match on both nodes (`az-301-ha-group`, Group ID `5`)
- [ ] Azure Load Balancer backend pool shows both nodes
- [ ] Health probes responding on ports 80 and 443

---

## Challenge! ⚔️

The External Load Balancer distributes incoming traffic across both FortiWeb nodes using
a hash-based algorithm. However, web application sessions often span multiple HTTP
requests — and all requests for the same session must reach the **same** FortiWeb node to
maintain session state. What Azure Load Balancer feature ensures this, and what are its
limitations in an Active-Active design?

---

## Lab 1 Complete! 🎉

### What You've Accomplished

✅ **Resource Group Created:**
- `redwood-app-protection-rg` in Canada Central

✅ **Virtual Network Deployed:**
- `vnet-app-protection` (10.0.0.0/16)
- Four subnets: `AzureBastionSubnet`, `external`, `internal`, `protected`

✅ **Azure Bastion Configured:**
- Browser-based secure access to private VMs — no public IPs required on app servers

✅ **FortiWeb HA Cluster Deployed via ARM Template:**
- `fweb-ha-vm1` and `fweb-ha-vm2` in Active-Active mode
- Azure External Load Balancer (`fweb-ha-loadbalance`) distributing traffic
- FortiFlex tokens activated automatically on first boot
- Both nodes in Availability Set for SLA coverage

✅ **Initial Configuration Completed:**
- Management access verified on both nodes
- FortiWeb `admin` password configured on both nodes
- HA cluster settings verified (`az-301-ha-group`, Group ID `5`)
- Azure Load Balancer backend pool confirmed healthy

### Architecture Review
```text
redwood-app-protection-rg (Canada Central)
│
├── vnet-app-protection (10.0.0.0/16)
│   ├── AzureBastionSubnet  (10.0.0.0/24)
│   ├── external            (10.0.1.0/24)  ← FortiWeb port1 (management + traffic)
│   ├── internal            (10.0.2.0/24)  ← FortiWeb port2 (to app servers)
│   └── protected           (10.0.3.0/24)  ← App servers [deployed in Lab 3]
│
├── bas-app-protection                      ← Azure Bastion
├── pip-bas-app-protection                  ← Bastion public IP
│
├── fweb-ha-loadbalance (External LB)       ← Ports 80/443 — fweb-ha-loadbalance-IP
│   └── FwbHaLBBackendAddrPool
│       ├── fweb-ha-vm1 (Node 1)           ← fweb-ha-nicPublic-IP1
│       └── fweb-ha-vm2 (Node 2)           ← fweb-ha-nicPublic-IP2
│
├── fweb-ha-availabilitySet
├── fweb-ha-securityGroup
└── [App servers — deployed in Lab 3]
```

### Key Takeaways

1. **ARM templates eliminate deployment errors:** Multi-resource HA deployments have many
   interdependencies. Templates ensure consistent, repeatable deployments that match
   Fortinet's reference architecture.

2. **FortiFlex via customData:** Tokens are injected at deployment time and activate
   automatically on first boot. No storage accounts, no manual GUI activation, no service
   principal required — this is the simplest BYOL deployment path for FortiWeb on Azure.

3. **Active-Active means both nodes are always working:** Unlike the Active-Passive
   FortiGate HA in AZ-102, there is no standby node sitting idle. Both FortiWeb nodes
   inspect traffic simultaneously — configuration replication (Lab 2) is what keeps
   them in sync.

4. **Three public IPs serve different purposes:** The load balancer IP (`fweb-ha-loadbalance-IP`)
   is the application VIP — clients connect here. The node IPs (`fweb-ha-nicPublic-IP1`,
   `fweb-ha-nicPublic-IP2`) are management-only — used only for GUI and SSH access.

5. **Azure Bastion protects your app servers:** The `protected` subnet has no internet
   exposure. Bastion provides browser-based SSH access to app servers in Lab 3 without
   requiring public IPs on those VMs.

### Validation Checklist

Before proceeding to Lab 2, verify:

**Infrastructure:**
- [ ] `redwood-app-protection-rg` exists in Canada Central
- [ ] `vnet-app-protection` has all four subnets with correct CIDRs
- [ ] Azure Bastion deployed (`bas-app-protection`)

**FortiWeb Cluster:**
- [ ] `fweb-ha-vm1` and `fweb-ha-vm2` both running
- [ ] GUI accessible on both nodes at port 8443
- [ ] FortiFlex licences valid on both nodes
- [ ] `admin` account password set on both nodes
- [ ] HA settings match: mode `Active-Active High Volume`, group `az-301-ha-group`, ID `5`
- [ ] Load Balancer backend pool shows both nodes healthy

### Next Steps

Ready for **Lab 2: FortiWeb Configuration Replication!**

In Lab 2, you will:

- Understand why Active-Active FortiWeb requires explicit configuration replication
- Designate Node 1 (`fweb-ha-vm1`) as the configuration **Server**
- Designate Node 2 (`fweb-ha-vm2`) as the configuration **Client**
- Trigger a manual sync and verify that Node 2 mirrors Node 1's configuration
- Establish the replication workflow you will follow after every configuration change
  in Labs 3, 4, and 5

---

## Troubleshooting Reference

### Issue: ARM Template Deployment Fails

Navigate to `redwood-app-protection-rg > Deployments`, click the failed deployment, and
read the **Error** section — it typically identifies the exact parameter causing the
failure.

**Common causes:**

- **VNet or subnet name mismatch:** The template parameters `vnetName`, `vnetSubnet1Name`,
  and `vnetSubnet2Name` must match exactly what you created in Part B — including case
- **Resource group mismatch:** Ensure `vnetResourceGroup` matches `redwood-app-protection-rg`
  exactly
- **Invalid FortiFlex token format:** Tokens are long alphanumeric strings — paste them
  rather than typing manually
- **VM SKU not available:** `Standard_F2s_v2` may not be available in all Azure
  subscriptions — check quota under **Subscriptions > Usage + quotas**

### Issue: Cannot Access FortiWeb GUI

1. Verify you are using `https://` and port `8443` (not 443 or 8080)
2. Accept the certificate warning — the self-signed certificate is expected
3. Verify the VM shows **Running** state (not **Starting** or **Deallocated**)
4. The NSG `fweb-ha-securityGroup` created by the template already allows port 8443
   inbound — if access still fails, verify no other NSG is applied to the subnet

### Issue: FortiFlex Licence Not Activating

1. Wait 5 minutes — first-boot customData processing takes time
2. Verify the FortiWeb VM has internet connectivity (outbound to Fortinet's servers)
3. Check that your FortiFlex token has not already been consumed by another VM
4. Navigate to **System > Status** and look for any licence error messages
5. If needed, activate manually via CLI console:
```
   execute forticloud-vm activate <your-token>
```

### Issue: HA Settings Not Showing Expected Values

The HA settings (`az-301-ha-group`, Group ID `5`) are configured via `customData` during
first boot. If they are missing:

1. Verify the VM completed its first boot successfully — check **Boot diagnostics** in
   the Azure Portal under the VM resource
2. Reboot the VM from the Azure Portal and wait 3–4 minutes
3. Reconnect to the GUI and recheck **System > High Availability > Settings**

### Issue: Load Balancer Shows Nodes as Unhealthy

1. Wait 3–5 minutes after deployment — nodes take time to fully boot and begin responding
   to health probes
2. Verify both VMs are in **Running** state
3. The template configures health probes on TCP ports 80 and 443 — verify FortiWeb is
   listening on these ports (it does by default)
4. Confirm the NSG `fweb-ha-securityGroup` has not been modified — it must allow inbound
   traffic on ports 80 and 443 from `168.63.129.16` (Azure's health probe source IP)

---

*Lab Guide Version 1.0 — January 2025*

*Next: [Lab 2 — FortiWeb Configuration Replication](/az-301-lab2/README.md)*
























## Part C: Prepare for ARM Template Deployment

Before deploying the FortiWeb cluster, you need three pieces of Azure identity information. The ARM template uses these to allow FortiWeb to call Azure APIs during HA operations (updating load balancer backend pools on failover).

### Step 6: Gather Required Azure Credentials

#### 6.1 Get Subscription ID and Tenant ID

1. In the Azure Portal search bar, type `subscriptions`
2. Click **Subscriptions**
3. Click on your active subscription
4. Copy and save the following values:

| Value | Where to Find It | Example Format |
|---|---|---|
| **Subscription ID** | Overview page, **Subscription ID** field | `cf72478e-c3b0-xxxx-xxxx-xxxxxxxxxxxx` |
| **Tenant ID** | Overview page, or navigate to **Microsoft Entra ID > Overview** | `942b80cd-1b14-xxxx-xxxx-xxxxxxxxxxxx` |

> [!TIP]
> Open a text editor (Notepad, VS Code) and paste both values there now. You will need them in Step 8.

#### 6.2 Get the App Registration (Service Principal) Credentials

The FortiWeb ARM template requires an Azure App Registration (service principal) with permissions to modify load balancer backend pools.

> [!NOTE]
> In a partner or customer deployment, you would create a dedicated App Registration with the **Network Contributor** role scoped to the resource group. For this workshop, the credentials are pre-provisioned in your Student Resources panel.

Locate and save the following from your Student Resources:

| Value | Description |
|---|---|
| **Application (client) ID** | The App Registration's unique identifier (RestApp ID in the template) |
| **Client secret** | The secret key for the App Registration (RestApp Secret in the template) |

### Validation

- [ ] Subscription ID copied
- [ ] Tenant ID copied
- [ ] App Registration client ID copied
- [ ] App Registration client secret copied

---













## Part D: Deploy the FortiWeb HA Cluster

### Step 7: Create the ARM Template Spec

#### 7.1 Navigate to Template Specs

1. In the Azure Portal search bar, type `template specs`
2. Click **Template specs** in the results
3. Click **+ Create template spec**

#### 7.2 Configure the Template Spec

1. In the **Basics** tab, configure:

| Setting | Value |
|---|---|
| **Subscription** | Your Azure subscription |
| **Resource group** | `redwood-app-protection-rg` |
| **Name** | `fortiweb-ha-template` |
| **Location** | `Canada Central` |
| **Version** | `1.0.0` |

2. Click **Next: Edit Template**

#### 7.3 Paste the ARM Template Content

1. In the template editor, select all existing content and delete it
2. Open the following URL in a new browser tab:
```
https://raw.githubusercontent.com/regisftm/AZ-301/refs/heads/main/arm-template/deploy_fwb_ha_v6.json
```

3. Select all content (`Ctrl+A`) and copy it (`Ctrl+C`)
4. Return to the Azure Portal template editor and paste the content (`Ctrl+V`)
5. Click **Review + create**
6. Click **Create**

> [!NOTE]
> The template spec is a reusable container for the ARM template. Creating it first means you can deploy the same template multiple times without re-pasting the JSON.

### Step 8: Deploy the FortiWeb HA Cluster

#### 8.1 Open the Template for Deployment

1. After creation, click on **fortiweb-ha-template** in the template spec list
2. Click **Deploy**

#### 8.2 Configure Basics

| Setting | Value |
|---|---|
| **Subscription** | Your Azure subscription |
| **Resource group** | `redwood-app-protection-rg` |
| **Region** | `Canada Central` |

#### 8.3 Configure Template Parameters

**Identity Parameters:**

| Parameter | Value |
|---|---|
| **Subscription Id** | Your Subscription ID (from Step 6.1) |
| **Tenant Id** | Your Tenant ID (from Step 6.1) |
| **Restapp Id** | App Registration client ID (from Step 6.2) |
| **Restapp Secret** | App Registration client secret (from Step 6.2) |

**VM Configuration:**

| Parameter | Value |
|---|---|
| **Resource Name Prefix** | `fwb` |
| **Vm Sku** | `Standard_F2s_v2` |
| **Availability Options** | `Availability Set` |
| **Accelerated Networking** | `false` |
| **Vm Admin Username** | `fortiuser` |
| **Vm Authentication Type** | `password` |
| **Vm Admin Password** | Create a strong password — save it securely |
| **Vm Ssh Public Key** | Leave blank (`-`) |
| **Vm Image Type** | `BYOL` |
| **Vm Image Version** | `latest` |
| **Vm Count** | `2` |

> [!WARNING]
> The admin username cannot be `admin` or `root` — Azure will reject these. Use `fortiuser` as shown above.

> [!WARNING]
> The password must meet at least 3 of these 4 conditions: lowercase characters, uppercase characters, a digit, a special character. Save this password — you will need it throughout the workshop.

**Network Configuration:**

| Parameter | Value |
|---|---|
| **Vnet New Or Existing** | `existing` |
| **Vnet Resource Group** | `[resourceGroup().name]` |
| **Vnet Name** | `vnet-app-protection` |
| **Vnet Address Prefix** | `10.0.0.0/16` |
| **Vnet Subnet1 Name** | `external` |
| **Vnet Subnet1 Prefix** | `10.0.1.0/24` |
| **Vnet Subnet2 Name** | `internal` |
| **Vnet Subnet2 Prefix** | `10.0.2.0/24` |

**HA Configuration:**

| Parameter | Value |
|---|---|
| **Load Balancer Type** | `Public` |
| **Fortiweb Ha Mode** | `active-active` |
| **Fortiweb Ha Group Name** | `fwb-ha-group` |
| **Fortiweb Ha Group Id** | `0` |
| **Fortiweb Ha Override** | `disable` |

**BYOL Licensing (FortiFlex):**

| Parameter | Value |
|---|---|
| **Storage Account Name** | Leave blank (`-`) |
| **Storage Access Key** | Leave blank (`-`) |
| **Storage License Container Name** | Leave blank (`-`) |

> [!NOTE]
> **FortiFlex Licensing:** This workshop uses FortiFlex tokens rather than traditional `.lic` files. Leave the storage fields blank — you will activate your FortiFlex tokens directly in the FortiWeb GUI after deployment in Part E.

#### 8.4 Deploy

1. Click **Review + create**
2. Review all parameters carefully — pay particular attention to the network parameters and credentials
3. Click **Create**

> [!NOTE]
> The ARM template deployment typically takes **8–12 minutes**. It is creating the following resources simultaneously: availability set, two FortiWeb VMs, four network interfaces, public load balancer, backend pool, health probes, and load balancing rules. This is a good time to refill your coffee. ☕

#### 8.5 Monitor Deployment Progress

1. Click the notification bell 🔔 at the top right of the portal to see deployment status
2. Alternatively, navigate to `redwood-app-protection-rg > Deployments` for detailed progress

### Validation

When deployment completes, verify the following resources exist in `redwood-app-protection-rg`:

- [ ] `fwb-vm0` — FortiWeb Node 1 (VM)
- [ ] `fwb-vm1` — FortiWeb Node 2 (VM)
- [ ] External Load Balancer with public IP
- [ ] Two network interfaces per VM
- [ ] Availability set containing both VMs

---

## Part E: Access FortiWeb and Activate Licensing

### Step 9: Locate the FortiWeb Public IP

#### 9.1 Find the Management IP

1. Navigate to `redwood-app-protection-rg`
2. Click on the **Public IP address** resource associated with `fwb-vm0`
3. Note the **IP address** value — this is your FortiWeb management IP

> [!TIP]
> The HA role (primary/secondary) of each node is visible under the VM's **Tags** tab — look for the `ha-role` tag.

### Step 10: Connect to the FortiWeb GUI

#### 10.1 Open the FortiWeb Management Interface

1. Open a new browser tab and navigate to:
```
https://<fwb-vm0-public-ip>:8443
```

2. You will see a certificate warning — this is expected (self-signed certificate)
3. Click **Advanced** → **Proceed** (or equivalent in your browser)

#### 10.2 Log In

| Field | Value |
|---|---|
| **Username** | `fortiuser` |
| **Password** | The VM admin password you set in Step 8.3 |

> [!NOTE]
> The `fortiuser` account is the VM operating system administrator, not the FortiWeb `admin` account. You will configure the FortiWeb `admin` account password in Step 11.

### Step 11: Configure the FortiWeb Admin Account

The FortiWeb `admin` account is separate from the VM `fortiuser` account. You must set the `admin` password via CLI before it can be used.

#### 11.1 Open the CLI Console

1. In the FortiWeb GUI, click the **CLI Console** icon in the top-right toolbar

#### 11.2 Set the Admin Password
```
config system admin
    edit "admin"
        set password "YourStrongPassword1!"
    next
end
```

> [!WARNING]
> Replace `YourStrongPassword1!` with a strong password of your choice. The `admin` account password **must not be empty** on Azure-deployed FortiWeb instances. Save this password — you will use the `admin` account for all configuration tasks in Labs 2 through 5.

### Step 12: Activate FortiFlex Licensing

#### 12.1 Apply FortiFlex Tokens — Node 1

1. In the FortiWeb GUI, navigate to **System > FortiGuard**
2. Under **License Information**, click **Activate License**
3. Enter your FortiFlex token for Node 1
4. Click **Activate**

#### 12.2 Apply FortiFlex Tokens — Node 2

1. Open a new browser tab and navigate to:
```
https://<fwb-vm1-public-ip>:8443
```

2. Log in with `fortiuser` credentials
3. Repeat Step 12.1 using the second FortiFlex token

#### 12.3 Verify Licence Status

1. On each node, navigate to **System > Status > Status**
2. Confirm the licence status shows as **Valid**

> [!NOTE]
> After licence activation, FortiWeb may prompt you to reboot. Allow the reboot — it takes approximately 2–3 minutes. The other node remains active and continues processing traffic during this time.

### Step 13: Verify HA Cluster Status

#### 13.1 Check HA Configuration

1. Navigate to **System > High Availability > Settings**
2. Verify:
   - **Mode:** Active-Active
   - **Group Name:** `fwb-ha-group`
   - **Group ID:** `0`
   - Both cluster members are listed

#### 13.2 Verify Load Balancer Health Probes

1. In the Azure Portal, navigate to the **Load balancer** resource in `redwood-app-protection-rg`
2. Click **Settings > Backend pools**
3. Verify both `fwb-vm0` and `fwb-vm1` are listed as backend pool members

> [!NOTE]
> If a backend pool member shows as **Unhealthy**, verify the FortiWeb node has completed its boot sequence and the licence is active. Health probe failures during initial deployment are common and typically resolve within 2–3 minutes.

### Validation

- [ ] FortiWeb GUI accessible at `https://<public-ip>:8443`
- [ ] Logged in successfully with `fortiuser` credentials
- [ ] `admin` account password configured via CLI
- [ ] FortiFlex tokens activated on both nodes
- [ ] HA status shows Active-Active with both members listed
- [ ] Azure Load Balancer backend pool shows both nodes as healthy

---

## Challenge! ⚔️

The External Load Balancer distributes incoming traffic across both FortiWeb nodes. But how does FortiWeb ensure that the response packet from the application server returns through the **same** FortiWeb node that received the client's request? What FortiWeb or Azure mechanism prevents asymmetric traffic flow in this Active-Active design?

---

## Lab 1 Complete! 🎉

### What You've Accomplished

✅ **Resource Group Created:**
- `redwood-app-protection-rg` in Canada Central

✅ **Virtual Network Deployed:**
- `vnet-app-protection` (10.0.0.0/16)
- Four subnets: Bastion, external, internal, protected

✅ **Azure Bastion Configured:**
- Browser-based secure access to private VMs — no public IPs required on app servers

✅ **FortiWeb HA Cluster Deployed:**
- Two nodes in Active-Active mode via ARM template
- Azure External Load Balancer distributing traffic
- Both nodes in Availability Set for SLA coverage

✅ **Initial Configuration Completed:**
- Management access verified
- FortiWeb `admin` password configured
- FortiFlex licensing activated on both nodes
- HA cluster health confirmed

### Architecture Review
```text
redwood-app-protection-rg (Canada Central)
│
├── vnet-app-protection (10.0.0.0/16)
│   ├── AzureBastionSubnet (10.0.0.0/24)
│   ├── external           (10.0.1.0/24)  ← FortiWeb port1
│   ├── internal           (10.0.2.0/24)  ← FortiWeb port2
│   └── protected          (10.0.3.0/24)  ← App servers [Lab 3]
│
├── bas-app-protection                     ← Azure Bastion
├── pip-bas-app-protection                 ← Bastion public IP
│
├── External Load Balancer                 ← Public IP, ports 80/443
│   └── Backend Pool
│       ├── fwb-vm0 (Node 1)
│       └── fwb-vm1 (Node 2)
│
└── [App servers — deployed in Lab 3]
```

### Key Takeaways

1. **ARM templates eliminate deployment errors:** Multi-resource HA deployments have many interdependencies. Templates ensure consistent, repeatable deployments that match Fortinet's reference architecture.

2. **Active-Active vs Active-Passive:** FortiWeb AA shares load across both nodes simultaneously. Both nodes process traffic — you get redundancy and capacity in one deployment. Configuration sync (Lab 2) keeps them aligned.

3. **FortiFlex simplifies BYOL:** No licence file management or storage accounts required. Tokens activate directly in the GUI and can be redeployed to different VM sizes as needs change.

4. **Three-subnet design reflects real deployments:** Separating external (internet-facing), internal (server-facing), and protected (application) subnets provides defence in depth and matches what you will encounter in customer environments.

5. **Azure Bastion is your secure management plane:** App servers in the `protected` subnet have no public IPs. Bastion provides browser-based SSH access without exposing those VMs to the internet.

### Validation Checklist

Before proceeding to Lab 2, verify:

**Infrastructure:**
- [ ] `redwood-app-protection-rg` exists in Canada Central
- [ ] `vnet-app-protection` has all four subnets with correct CIDRs
- [ ] Azure Bastion deployed (`bas-app-protection`)

**FortiWeb Cluster:**
- [ ] Both `fwb-vm0` and `fwb-vm1` running
- [ ] GUI accessible on HTTPS port 8443
- [ ] `admin` account password set
- [ ] FortiFlex tokens activated on both nodes
- [ ] HA mode shows Active-Active
- [ ] Load Balancer backend pool shows both nodes healthy

### Next Steps

Ready for **Lab 2: FortiWeb Configuration Replication!**

In Lab 2, you will:

- Understand the difference between FortiWeb HA sync and configuration replication
- Configure one FortiWeb node as the configuration **Server** (source)
- Configure the second node as the configuration **Client** (target)
- Initiate a manual sync and verify configuration propagation
- Establish the workflow for keeping both nodes aligned throughout the workshop

---

## Troubleshooting Reference

### Issue: ARM Template Deployment Fails

**Common causes and solutions:**

- **Incorrect Subscription/Tenant ID:** Copy directly from the Azure Portal — do not type manually
- **App Registration lacks permissions:** The service principal needs **Network Contributor** role on the resource group
- **VNet name mismatch:** Verify `vnet-app-protection` exists before deployment
- **Subnet name case mismatch:** The template expects `external` and `internal` (lowercase) — verify your subnet names match exactly

Navigate to `redwood-app-protection-rg > Deployments`, click the failed deployment, and read the **Error** section — it typically identifies the exact parameter causing the failure. Delete any partially created resources, correct the parameter, and redeploy.

### Issue: Cannot Access FortiWeb GUI

**Checklist:**

1. Use `https://` not `http://` and port `8443`
2. Accept the certificate warning (self-signed cert is expected)
3. Verify the VM is in **Running** state, not **Starting**
4. Check that the NSG on the `external` subnet allows TCP 8443 inbound from your IP
5. Verify no NSG is blocking the Azure Load Balancer health probe source (`168.63.129.16`)

### Issue: Load Balancer Shows Nodes as Unhealthy

**Checklist:**

1. Wait 2–3 minutes — nodes take time to fully boot and register
2. Verify both FortiWeb VMs are in **Running** state
3. Verify the health probe port matches FortiWeb's configured management port
4. Confirm no NSG is blocking `168.63.129.16` (Azure's health probe source IP)

### Issue: FortiFlex Token Activation Fails

**Checklist:**

1. Verify the token has not already been used on another VM
2. Ensure internet connectivity from the FortiWeb node (it must reach Fortinet's licensing servers)
3. Check `System > Status` for any licence-related error messages
4. Try activating via CLI: `execute forticloud-vm activate <token>`

### Issue: HA Cluster Shows Only One Member

**Checklist:**

1. Verify both VMs are running
2. Check HA settings on both nodes — Group Name and Group ID must match exactly
3. Verify the `internal` subnet has connectivity between both nodes (HA heartbeat uses port2)
4. Check FortiWeb event logs: **Log > Event Log > HA Events**

---

*Lab Guide Version 1.0 — January 2025*

*Next: [Lab 2 — FortiWeb Configuration Replication](/az-301-lab2/README.md)*