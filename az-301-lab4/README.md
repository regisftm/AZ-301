# Lab 4: WAF Protection and Attack Simulation

## Lab Overview

**Duration:** 35 minutes  
**Prerequisites:** Completed Labs 1, 2, and 3 — application accessible at the load balancer public IP, SQL injection payload confirmed to reach the application unblocked

### Objective

Enable FortiWeb WAF protection on the Redwood Industries application by attaching a protection profile to the existing Server Policy. You will explore FortiWeb's built-in protection profiles, attach the Inline Standard Protection profile to `policy-redwood-app`, and replay the same attacks from Lab 3 — this time observing FortiWeb block them before they reach the application. You will then review the attack logs to understand what FortiWeb detected and why.

### What You'll Build

By the end of this lab, you will have:

- ✅ Explored FortiWeb's built-in Web Protection Profiles
- ✅ Attached **Inline Standard Protection** to `policy-redwood-app`
- ✅ Confirmed SQL Injection attacks are blocked by FortiWeb
- ✅ Confirmed XSS attacks are blocked by FortiWeb
- ✅ Confirmed Path Traversal attacks are blocked by FortiWeb
- ✅ Confirmed Command Injection attacks are blocked by FortiWeb
- ✅ Reviewed attack logs and identified FortiWeb's detection signatures
- ✅ Configuration replicated to `fweb-ha-vm2`

### Architecture

After Lab 4:

```text
Internet
    │
    ▼
External Load Balancer (fweb-ha-loadbalance-IP)
    │
    ├──────────────────────────┐
    ▼                          ▼
fweb-ha-vm1                fweb-ha-vm2
    │                          │
    ▼                          ▼
policy-redwood-app         policy-redwood-app
    │  ← Inline Standard        │  ← Inline Standard
    │     Protection            │     Protection
    │                          │
    ▼                          ▼
pool-redwood-app ──────────────┘
    │
    ├── app-server-1 ✅ Protected
    └── app-server-2 ✅ Protected

Attack traffic → BLOCKED at FortiWeb ❌
Clean traffic  → FORWARDED to app servers ✅
```

### Business Context

**Redwood Industries' Requirement:**

With traffic flowing through FortiWeb, the security team is ready to enable active protection. The Lab 3 baseline test confirmed that attacks reach the application unimpeded when no WAF profile is attached. The security team now needs to demonstrate to Redwood's compliance officer that the OWASP Top 10 attack categories are being actively detected and blocked before they reach the application servers.

**What Changes in This Lab:**

A single configuration change — attaching a protection profile to `policy-redwood-app` — transforms FortiWeb from a transparent proxy into an active WAF. No infrastructure
changes, no new resources, no downtime. This is the power of the FortiWeb policy model: protection is layered on top of an already-working traffic path.

---

## Understanding FortiWeb Protection Profiles

### What is a Web Protection Profile?

A Web Protection Profile is a collection of security rules and settings that FortiWeb applies to HTTP/HTTPS traffic matching a Server Policy. It defines:

- Which **attack signatures** to detect (SQL injection, XSS, command injection, etc.)
- The **action** to take when a signature matches (Alert, Alert and Deny, Block Period)
- Additional protections such as HTTP protocol validation, bot mitigation, and rate limiting

### Built-in Profiles

FortiWeb ships with several pre-configured protection profiles. The two most relevant for this workshop are:

| Profile | Description | Use Case |
| --- | --- | --- |
| `Inline Standard Protection` | Comprehensive signature set covering OWASP Top 10, HTTP protocol validation, and common web attacks. Action: Alert & Deny | Production web applications requiring broad protection |
| `Inline Extended Protection` | Expanded signature set with additional rules for specific technologies and frameworks. Higher false-positive risk | Applications requiring deep inspection |

> [!NOTE]
> In **Inline** mode, FortiWeb sits directly in the traffic path as a reverse proxy.  
> When a rule matches, FortiWeb can block the request immediately — the application
> server never sees the malicious payload. This is the mode configured throughout this
> workshop.

### What Inline Standard Protection Covers

The **Inline Standard Protection** profile used in this lab provides detection and
blocking for:

- **SQL Injection** — attempts to manipulate database queries via user input
- **Cross-Site Scripting (XSS)** — injection of malicious scripts into web content
- **Path Traversal** — attempts to access files outside the web root
- **Command Injection** — embedding OS commands in HTTP parameters
- **HTTP Protocol Violations** — malformed requests that exploit parser weaknesses
- **Known attack tools** — scanners, crawlers, and exploit frameworks

---

## Part A: Explore the Built-in Protection Profiles

### Step 1: Review the Inline Standard Protection Profile

Before attaching the profile, take a few minutes to understand what it contains. This
context will make the attack log review in Part D more meaningful.

#### 1.1 Navigate to Web Protection Profiles

1. Connect to `fweb-ha-vm1` at `https://<fweb-ha-nic-pip1>:8443`
2. Log in with your `fortiuser` credentials
3. Navigate to **Policy > Web Protection Profile**
4. Locate **Inline Standard Protection** in the list and double-click on it

#### 1.2 Explore the Profile Components

The profile is a container of individual protection rule groups. Review the following
sections:

**Signature Protection:**

1. Scroll to the **Standard Protection** section
2. Click on the pencil icon in front of Signatures.
3. The **Signature Policy Standard Protection 👍** includes SQL Injection, XSS, Generic Attacks, and others
4. Note the **Action** column — **Alert & Deny** means FortiWeb blocks the request and logs the event
5. Click on **Return**
6. Click on **Return**

> [!NOTE]
> You are viewing a **read-only** built-in profile — you cannot modify it directly. In a production environment, you would clone this profile and customise it for your application's specific requirements. For this workshop, the built-in profile provides all the protection needed to demonstrate WAF capabilities.

#### 1.3 Note the Signature Count

1. Navigate to **Web Protection > Known Attacks > Signature** to view the full signature database.
2. Double-click the **Standard Protection** signature in the **Predefined** list of signatures.
3. These are the signatures enabled in the profile. To explore more, click on **Signature Details**
4. Explore the number of signatures available across all categories in the **Dictionaires**.

> [!TIP]
> FortiWeb's signature database is updated regularly via FortiGuard. The signatures
> your deployment uses reflect the database version at the time of your FortiFlex
> licence activation. Navigate to **System > FortiGuard** to verify your signature
> database is current.

### Validation

- [x] Reviewed **Inline Standard Protection** profile contents
- [x] Noted signature categories and actions (Alert & Deny)
- [x] Understood the difference between built-in and custom profiles

---

## Part B: Attach the Protection Profile

### Step 2: Apply Inline Standard Protection to the Server Policy

#### 2.1 Navigate to the Server Policy

1. Navigate to **Policy > Server Policy**
2. Double-click on `policy-redwood-app` to edit it

#### 2.2 Attach the Protection Profile

1. Locate the **Web Protection Profile** field — currently set to `— None —`
2. Click the dropdown and select **Inline Standard Protection**
3. Confirm the following settings remain unchanged:

   | Setting | Value |
   | --- | --- |
   | **Name** | `policy-redwood-app` |
   | **Virtual Server** | `vs-redwood-app` |
   | **Server Pool** | `pool-redwood-app` |
   | **HTTP Service** | `HTTP` |
   | **Web Protection Profile** | `Inline Standard Protection` ✅ |

4. Click **OK**

> [!NOTE]
> The protection profile takes effect **immediately** after saving — no restart or commit required. Requests arriving at FortiWeb from this point forward are inspected against the Inline Standard Protection signature set.

#### 2.3 Verify the Policy Status

1. Navigate back to **Policy > Server Policy**
2. Confirm `policy-redwood-app` still shows **Status: Running** (green link symbol)
3. Confirm the **Profile** column now shows `Inline Standard Protection`

### Step 3: Sync Configuration to `fweb-ha-vm2`

#### 3.1 Verify Replication on Node 2

1. Open a new browser tab and connect to `fweb-ha-vm2` at
   `https://<fweb-ha-nic-pip2>:8443`
2. Log in with your `fortiuser` credentials
3. Navigate to **Policy > Server Policy**
4. Click on `policy-redwood-app`
5. Confirm **Web Protection Profile** shows `Inline Standard Protection`

### Validation

- [x] `Inline Standard Protection` attached to `policy-redwood-app` on `fweb-ha-vm1`
- [x] Policy status remains **Running** after profile attachment
- [x] `Inline Standard Protection` confirmed on `policy-redwood-app` on `fweb-ha-vm2`

---

## Part C: Attack Simulation — Blocked by FortiWeb

All attacks in this section use the same load balancer public IP you tested in Lab 3.  
The key difference: FortiWeb now inspects every request before forwarding it.

### Step 4: SQL Injection — Now Blocked

#### 4.1 Replay the SQL Injection Attack

1. Open a browser tab and navigate to:

   ```http
   http://<fweb-ha-loadbalance-IP>/?name=' OR 'x'='x
   ```

1. In Lab 3, the page turned dark red and showed the attack details.
   **What do you see now?**

#### 4.2 Expected Result

FortiWeb intercepts the request and returns a block page before it reaches the
application server:

Web Page Blocked!
The page cannot be displayed. Please contact the administrator for additional information.

> [!NOTE]
> The exact appearance of the block page depends on your FortiWeb firmware version and
> whether a custom block page has been configured. The key indicator is that the
> **application's dark red attack state page is not shown** — the request never reached
> the application server.

#### 4.3 Verify the Attack was Logged

1. On `fweb-ha-vm1`, navigate to **Log&Report > Log Access > Attack**
2. You should see a new log entry with:

   | Field | Expected Value |
   | --- | --- |
   | **Attack Type** | `SQL Injection` or similar |
   | **Action** | `Alert & Deny` |
   | **Source IP** | Your public IP address |
   | **URL** | `/?name=' OR 'x'='x` |
   | **Policy** | `policy-redwood-app` |

> [!TIP]
> Double-click on the log entry to expand the full details. The **Signature ID** and **Signature Subclass Type** fields identify exactly which rule triggered. This information is valuable when tuning protection profiles to reduce false positives.

>[!TIP]
>If you can't find the log on `fweb-ha-vm1`, try the `fweb-ha-vm2`. This is where FortiAnalyzer becomes handy 😜

### Step 5: Cross-Site Scripting (XSS) — Blocked

#### 5.1 Attempt an XSS Attack

1. In your browser, navigate to:

   ```http
   http://<fweb-ha-loadbalance-IP>/?name=<script>alert(1)</script>
   ```

> [!NOTE]
> Some browsers URL-encode angle brackets automatically. If the attack page renders normally (not blocked), try copying the URL directly into the browser's address bar rather than clicking a link, or use a different browser.

#### 5.2 Expected Result

FortiWeb blocks the request and returns the block page. The application's attack state
page is not shown — the XSS payload never reached the server.

#### 5.3 Verify in Attack Log

1. Navigate to **Log&Report > Log Access > Attack**
2. Confirm a new entry for the XSS attempt with **Action: Alert_Deny**

### Step 6: Path Traversal — Blocked

#### 6.1 Attempt a Path Traversal Attack

1. In your browser, navigate to:

   ```http
   http://<fweb-ha-loadbalance-IP>/?file=../../etc/passwd
   ```

#### 6.2 Expected Result

FortiWeb blocks the request. The application server does not receive it.

#### 6.3 Verify in Attack Log

1. Navigate to **Log&Report > Log Access > Attack**
2. Confirm a new entry for the Path Traversal attempt

### Step 7: Command Injection — Blocked

#### 7.1 Attempt a Command Injection Attack

1. In your browser, navigate to:

   ```http
   http://<fweb-ha-loadbalance-IP>/?cmd=;cat /etc/shadow
   ```

#### 7.2 Expected Result

FortiWeb blocks the request before it reaches the application server.

#### 7.3 Verify in Attack Log

1. Navigate to **Log&Report > Log Access > Attack**
2. Confirm all four attack types now appear in the log

### Step 8: Confirm Clean Traffic Still Works

Blocking attacks is only half the job — confirming that legitimate traffic is
unaffected is equally important.

#### 8.1 Access the Application Normally

1. In your browser, navigate to:

   ```http
   http://<fweb-ha-loadbalance-IP>
   ```

2. The application should load normally — coloured gradient background, server name, **Service Online** badge

#### 8.2 Verify Clean Traffic in Logs

1. Navigate to **Log&Report > Log Access > Traffic**
2. Confirm traffic log entries exist for the clean requests
3. Note that clean traffic shows **Action: Accept** — FortiWeb forwarded these
   requests without intervention

### Validation

- [x] SQL Injection attack blocked — block page returned, not application attack state
- [x] XSS attack blocked by FortiWeb
- [x] Path Traversal attack blocked by FortiWeb
- [x] Command Injection attack blocked by FortiWeb
- [x] All four attack types appear in the attack log with **Alert & Deny** action
- [x] Clean traffic loads normally — application renders correctly
- [x] Clean traffic shows *success** Status in traffic log

---

## Part D: Analyse the Attack Logs

### Step 9: Review Attack Log Details

#### 9.1 Open the Attack Log

1. Navigate to **Log&Report > Log Access > Attack**
2. You should see entries for all four attack types tested in Part C

#### 9.2 Examine a Log Entry in Detail

1. Click on the **SQL Injection** log entry to expand it
2. Review the following fields:

   | Field | Description |
   | --- | --- |
   | **Date / Time** | When the attack was detected |
   | **Source IP** | The IP address of the attacker (your public IP) |
   | **Method** | `get` — SQL injection via URL parameter |
   | **URL** | The full request URL including the malicious payload |
   | **Signature ID** | The unique identifier of the matching rule |
   | **Signature Subclass Type** | Human-readable description of the signature |
   | **Threat Level** | `Critical` for SQL injection |
   | **Action** | `Alert & Deny` — request was blocked |
   | **Policy** | `policy-redwood-app` |

> [!NOTE]
> The **Signature ID** is particularly useful when working with Fortinet support or when a rule produces false positives. You can use the ID to look up the exact rule in FortiGuard's signature database and create an exception if needed.

#### 9.3 Compare Attack Severity Levels

Threat levels are used to categorize the severity of potential security threats. Here are the possible threat levels:

- **Informational**: This level indicates that the event is for informational purposes only and does not pose any threat.
- **Low**: A low threat level suggests minimal risk, often requiring monitoring but not immediate action.
- **Moderate**: This level indicates a moderate risk that may require attention and potential action to mitigate.
- **Substantial**: A substantial threat level signifies a significant risk that should be addressed promptly to prevent potential security breaches.
- **Severe**: This level indicates a high risk that requires immediate action to protect the network or system.
- **Critical**: The critical threat level represents the highest risk, necessitating urgent and comprehensive measures to counteract the threat.

These threat levels help in prioritizing responses and managing security measures effectively. They are often visually represented with color codes to quickly convey the severity of the threat.

1. Review all four attack log entries
2. Note the **Severity** assigned to each:

   | Attack Type | Typical Severity |
   | --- | --- |
   | SQL Injection | Severe |
   | XSS | Severe |
   | Path Traversal | Substantial |
   | Command Injection | Substantial |

> [!NOTE]
> Severity levels are defined by the Fortinet signature database based on the potential impact of a successful exploit. SQL injection and Cross Site Scripting rank highest because a successful attack can result in full database or system compromise.

### Step 10: Explore FortiView

FortiView provides a graphical summary of traffic and threat activity. It is particularly useful for demonstrating FortiWeb's value to stakeholders and for rapid incident triage.

#### 10.1 Open FortiView

1. Navigate to **Dashboard**

#### 10.2 Explore the Threat Map

1. Click on **FortiView Threat Map**
2. Observe the geographic source of detected attacks — your test attacks will appear from your current location

#### 10.3 Review the Threats View

1. Click on **Threats**
2. You should see the four attack categories from Part C ranked by Threat Level
3. Note the breakdown by Threat, Threat Level, Threat Score, Action and Service

> [!TIP]
> In a real customer demonstration, FortiView is where you start. The visual threat map and top threats summary immediately communicate the volume and variety of attacks being blocked — making the value of FortiWeb tangible to non-technical stakeholders.

### Validation

- [x] Attack log entries reviewed for all four attack types
- [x] Signature ID and Name noted for at least one entry
- [x] Severity levels compared across attack types
- [x] FortiView reviewed — attack statistics visible

---

## Lab 4 Complete! 🎉

### What You've Accomplished

✅ **Protection Profile Explored:**

- Reviewed **Inline Standard Protection** signature categories and actions
- Understood the difference between Alert-only and Alert & Deny actions

✅ **WAF Protection Enabled:**

- `Inline Standard Protection` attached to `policy-redwood-app`
- Configuration replicated to `fweb-ha-vm2` — both nodes actively protecting

✅ **All Four Attack Types Blocked:**

- SQL Injection — blocked before reaching application server
- Cross-Site Scripting (XSS) — blocked before reaching application server
- Path Traversal — blocked before reaching application server
- Command Injection — blocked before reaching application server

✅ **Legitimate Traffic Unaffected:**

- Application loads correctly for clean requests
- Round Robin load balancing continues to function
- No disruption to normal application behaviour

✅ **Attack Logs Reviewed:**

- All attacks appear in the attack log with full details
- Signature IDs, severity levels, and actions confirmed
- FortiView traffic and threat statistics explored

### Architecture Review

```text
redwood-app-protection-rg (Canada Central)
│
├── vnet-app-protection (10.0.0.0/16)
│   ├── AzureBastionSubnet  (10.0.0.0/26)
│   ├── external            (10.0.1.0/24)
│   │   ├── fweb-ha-vm1 port1
│   │   └── fweb-ha-vm2 port1
│   ├── internal            (10.0.2.0/24)
│   │   ├── fweb-ha-vm1 port2
│   │   └── fweb-ha-vm2 port2
│   └── protected           (10.0.3.0/24)
│       ├── app-server-1 ✅ WAF-protected
│       └── app-server-2 ✅ WAF-protected
│
├── fweb-ha-loadbalance (External LB)
│   │
│   ├── fweb-ha-vm1
│   │   └── policy-redwood-app
│   │       ├── WAF: Inline Standard Protection ✅
│   │       ├── vs-redwood-app  (port1 IP)
│   │       └── pool-redwood-app
│   │           ├── app-server-1:80
│   │           └── app-server-2:80
│   │
│   └── fweb-ha-vm2 (identical — replicated)
│
Attack traffic ❌ → Blocked at FortiWeb → Logged in attack log
Clean traffic  ✅ → Forwarded to app servers → Logged in traffic log
```

### Key Takeaways

1. **A single policy change enables WAF protection:** Attaching a protection profile to an existing Server Policy requires no infrastructure changes, no downtime, and no reconfiguration of the network. FortiWeb's object model makes enabling and disabling protection surgical and reversible.

2. **Built-in profiles provide immediate OWASP Top 10 coverage:** The Inline Standard Protection profile covers the most critical web attack categories out of the box. For most applications, this profile provides a strong production starting point that can be refined over time.

3. **Blocking attacks is only meaningful if logs capture them:** The attack log is the evidence trail. Every blocked request is recorded with the source IP, payload, matched signature, and action taken — essential for incident response, compliance reporting, and profile tuning.

4. **Clean traffic must remain unaffected:** A WAF that blocks legitimate users is as damaging as no WAF at all. Verifying clean traffic after enabling protection is a mandatory step — not an optional one.

5. **Both nodes must have identical protection:** The configuration replication workflow (change on Node 1, verify on Node 2) is especially critical when attaching protection profiles. A user whose session is load-balanced to an unprotected node receives no WAF coverage.

### Validation Checklist

Before closing out the workshop, verify:

**FortiWeb Configuration (both nodes):**

- [x] `Inline Standard Protection` attached to `policy-redwood-app` on both nodes
- [x] Policy status **Running** on both nodes

**Attack Blocking:**

- [x] SQL Injection blocked — block page returned
- [x] XSS blocked — block page returned
- [x] Path Traversal blocked — block page returned
- [x] Command Injection blocked — block page returned

**Logs:**

- [x] All four attack types in attack log with **Alert & Deny** action
- [x] Clean traffic visible in traffic log with **success** status

---

## Workshop Complete! 🏆

You have completed the full AZ-301 workshop. Here is a summary of what Redwood
Industries now has running in Azure:

| Component | Resource | Status |
| --- | --- | --- |
| Resource Group | `redwood-app-protection-rg` | ✅ |
| Virtual Network | `vnet-app-protection` (10.0.0.0/16) | ✅ |
| Azure Bastion | `bas-app-protection` | ✅ |
| NAT Gateway | `nat-app-protection` | ✅ |
| FortiWeb Node 1 | `fweb-ha-vm1` — Config Server | ✅ |
| FortiWeb Node 2 | `fweb-ha-vm2` — Config Client | ✅ |
| External Load Balancer | `fweb-ha-loadbalance` | ✅ |
| Application Server 1 | `app-server-1` — protected subnet | ✅ |
| Application Server 2 | `app-server-2` — protected subnet | ✅ |
| WAF Protection | Inline Standard Protection — all OWASP Top 10 categories | ✅ |

### Clean Up

> [!CAUTION]
> This workshop deploys several Azure resources that incur ongoing costs when left running. **Delete the resource group immediately after the workshop** to avoid unexpected charges. Estimated cost for the 3-hour workshop if deleted promptly: **$5–10 CAD**.

To delete all workshop resources in one step:

1. Navigate to `redwood-app-protection-rg` in the Azure Portal
2. Click **Delete resource group**
3. Type `redwood-app-protection-rg` to confirm
4. Click **Delete**
5. Confirm **Delete**

All resources created during this workshop — VMs, load balancer, VNet, Bastion, NAT Gateway, public IPs, NICs, NSGs, and availability set — are deleted in a single operation.

---

## Troubleshooting Reference

### Issue: Attacks Not Being Blocked After Attaching Profile

1. Verify the profile is attached on **both** `fweb-ha-vm1` and `fweb-ha-vm2` —
   navigate to **Policy > Server Policy > policy-redwood-app** on each node
2. Verify the policy status is **Running** — not **Disable**
3. Clear your browser cache and try again — a cached response may show the old (unblocked) page

### Issue: Legitimate Traffic Being Blocked (False Positives)

1. Navigate to **Log&Report > Log Access > Attack** and identify the Signature ID that triggered
2. Navigate to **Web Protection > Signature**
3. Search for the Signature ID
4. You can change the **Action** for that specific signature to **Alert** (log only, do not block) or create a **Signature Exception** scoped to the specific URL and parameter

### Issue: Attack Log is Empty

1. Verify traffic log is enabled on the policy (configured in Lab 3, Step 6.4)
2. Verify you are checking the correct node — the log entry appears on whichever node the load balancer sent the request to
3. Check both `fweb-ha-vm1` and `fweb-ha-vm2` attack logs
4. Ensure you waited 30–60 seconds after the attack attempt before checking — log entries may have a brief write delay

### Issue: Block Page Not Appearing — Application Still Renders

1. Verify the protection profile is **Inline Standard Protection** — not a custom profile with Alert-only actions
2. Verify **Action** in the profile is **Alert & Deny** — not **Alert** alone
3. Try the attack from a different browser or incognito window — your browser may be serving a cached version of the application page
4. Check the attack log — if the attack appears in the log with **Alert & Deny**, FortiWeb is blocking correctly but the block page may be rendering behind a cached browser response

---

*Lab Guide Version 1.0 — April 2026*

*AZ-301 Workshop Complete!*
