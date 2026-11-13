# Getting Started & Accessing the Sandbox

This guide covers account onboarding, multi-factor authentication setup, web access via **Nebari**, and command-line head node access via **Okta Privileged Access (OPA)**.

> **Need Help?**  
> Contact the Sandbox Helpdesk at **sandbox.helpdesk@noaa.gov** for onboarding assistance, IP allowlisting, or head node assignment.

---

## 1. Account Onboarding & Prerequisites

Before logging into the Sandbox for the first time, complete the following onboarding steps:

1. **Request a Project & Account:** Email **sandbox.helpdesk@noaa.gov** and submit the **Sandbox Google Project Request Form**. The outreach team will follow up to discuss your project requirements and provision your accounts.
2. **Track Onboarding Status:** Monitor the status of your user account and head node provisioning on the **NOS Sandbox User Onboarding Status Dashboard**.
3. **Network & IP Allowlisting (External / Non-NOAA Users):** To reach the Nebari portal (`https://coastalsandbox.nos.noaa.gov`), you must connect from a NOAA network, the NOAA VPN, or an allowlisted IP address. If you are an external partner, email your public IP address (or organizational network CIDR range from your IT administrator) to **sandbox.helpdesk@noaa.gov**.

---

## 2. Okta Account & MFA Setup

Depending on your affiliation and compute needs, you will use one or both of the following identity configurations:

* **NOAA Internal Users (`@noaa.gov`):** Authenticate to Nebari via **NOAA ICAM** using your NOAA CAC card. If your project requires direct terminal access to a head node, you will also need an **Okta Privileged Access (OPA)** account.
* **External / Non-NOAA Users:** Require an **Okta** account for Nebari access, and an **Okta Privileged Access (OPA)** account if accessing a head node.

### Activating Your Okta Account & Okta Verify (MFA)

Once registered by the Sandbox team, you will receive an activation email from `noreply@okta.com` (the activation link expires in **7 days**). All users with a Sandbox Okta account must configure **Okta Verify** before logging in:

1. Install the **Okta Verify** app on your mobile device *(note: Okta Verify can only be registered on one device at a time)*.
2. On your computer, navigate to [https://noaa.okta.com](https://noaa.okta.com) and sign in with your credentials.
3. When prompted to enroll in Okta Verify, open the mobile app and scan the QR barcode displayed on your screen.
4. For future logins, approve the push notification on your mobile device or enter the numeric MFA code from the Okta Verify app.

---

## 3. Sandbox Access Pathways

The Sandbox supports two primary access pathways depending on your workflow:

```text
                        +-----------------------+
                        |     Sandbox User      |
                        +-----------+-----------+
                                    |
                   +----------------+----------------+
                   |                                 |
                   v                                 v
    +------------------------------+   +---------------------------+
    |   Web UI / Python Notebooks  |   | Local Workstation Terminal|
    |  (coastalsandbox.nos.noaa.gov)|   |   (Local Okta OPA CLI)    |
    +--------------+---------------+   +-------------+-------------+
                   |                                 |
          +--------+--------+                        |
          |                 |                        |
          v                 v                        |
   +-------------+   +-------------+                 |
   |  NOAA ICAM  |   |  NOAA Okta  |                 |
   | (@noaa.gov) |   | (External)  |                 |
   +------+------+   +------+------+                 |
          |                 |                        |
          +--------+--------+                        |
                   |                                 |
                   v                                 |
    +------------------------------+                 |
    |   Nebari JupyterLab / IDE    |                 |
    | (Pre-installed Okta OPA CLI) |                 |
    +--------------+---------------+                 |
                   |                                 |
                   +----------------+----------------+
                                    |
                                    v
                     +-----------------------------+
                     |  sft enroll  -->  sft ssh   |
                     +--------------+--------------+
                                    |
                                    v
                     +-----------------------------+
                     |     Sandbox Head Node       |
                     | (/save, /com, Cloudflow)    |
                     +-----------------------------+
```

## 4. Logging into the Sandbox (4a: Nebari vs. 4b: Okta OPA)

Depending on your workflow, use **4a** for web-based interactive work in Nebari or **4b** for command-line access to a head node via Okta Privileged Access (OPA).

---

### 4a. Logging into Nebari (Web Interface)

1. Ensure you are connected to a NOAA network, the NOAA VPN, or an allowlisted IP address.
2. Navigate to **[https://coastalsandbox.nos.noaa.gov](https://coastalsandbox.nos.noaa.gov)**.
3. Click **Sign in with Keycloak**.
4. Select your external Identity Provider at the bottom of the login card:
   * **NOAA ICAM:** For users with a `@noaa.gov` email address and CAC card.
   * **NOAA Okta:** For all external/non-NOAA Sandbox users.

> **Important:** Do **not** enter credentials into the `Username` field or click the blue `Sign In` button on the Keycloak form—local password login is disabled. Always use the **NOAA ICAM** or **NOAA Okta** buttons below it.

*(Recommended Visual: Add sanitized screenshots of the Keycloak login buttons to `./images/nebari_keycloak_login.png`.)*

---

### 4b. Logging into a Head Node via Okta OPA (CLI)

#### Prerequisites
* The Sandbox team has deployed a head node for your project or assigned you to an existing head node in Okta OPA.
* You have completed Okta Account and Okta Verify MFA setup.

#### Choosing Where to Run the OPA Client (`sft`)
You can run the Okta Privileged Access (`sft`) CLI from either:
* **Option A — Inside Nebari (No Local Install Required):** Open a Terminal in Nebari JupyterLab, where the `sft` client is pre-installed and configured.
* **Option B — Your Local Workstation:** Install the Okta Privileged Access (`sft`) client for your operating system following the official Okta documentation.

#### Step 1: One-Time Client Enrollment (`sft enroll`)

Enroll your `sft` client with the Sandbox OPA team. You only need to run this once per profile/machine:

```bash
sft enroll --url [https://noaa.pam.okta.com/team](https://noaa.pam.okta.com/team) nos-coastal-modeling-cloud-sandbox
```

Because head nodes and Nebari terminals do not have a local desktop browser, `sft` will print an approval URL in your terminal:

```text
Failed to open default browser: tried to open "[https://noaa.pam.okta.com/r/clients/](https://noaa.pam.okta.com/r/clients/)<id>/approve" - no display environment found
Please open this URL in your browser to complete enrollment:
[https://noaa.pam.okta.com/r/clients/](https://noaa.pam.okta.com/r/clients/)<id>/approve
Waiting on browser...
```

1. Copy and open the approval URL in your local web browser.
2. Select the **`nos-coastal-modeling-cloud-sandbox`** team (if it does not appear automatically, type `nos-coastal-modeling-cloud-sandbox` or your project team name into the **Add a team** box and submit).
3. Once approved in the browser, your terminal will confirm:

```text
Browser enrollment completed successfully.
Enrolled Device: user <your.username> team nos-coastal-modeling-cloud-sandbox
```

#### Step 2: List Assigned Servers (`sft list-servers`)

To view the head nodes assigned to your account, run:

```bash
sft list-servers
```

*(If your OPA session token has expired, `sft` will prompt you with a browser authentication link before displaying the server table.)*

Example output:

```text
HOSTNAME        OS_TYPE   PROJECT_NAME   ID                                     ACCESS_ADDRESS   Login_Users
head_node_164   linux                    2301f4c1-5fed-4c2d-83ed-67c5a64cbccb   10.26.36.164
```

#### Step 3: Connect via SSH (`sft ssh`)

Connect to your head node using its `HOSTNAME` from the list above:

```bash
sft ssh <HOSTNAME>
# Example:
sft ssh head_node_164
```

You will be logged into the head node with a personal user account and home directory.

> **First-Time Login Action Required:**  
> After your very first successful `sft ssh` login, email **sandbox.helpdesk@noaa.gov** so the Sandbox team can finalize your workspace directories (`/save/$USER` and `/com/$USER`) and confirm your assigned machine name.

---

## 5. Initial Workspace Setup on the Head Node

Once logged into your head node, verify your workspace directories and avoid working inside `/home` (which is a small, ephemeral root volume):

```bash
# Navigate to your persistent EFS workspace and clone the repository
cd /save
mkdir -p $USER
cd $USER
git clone [https://github.com/ioos/Cloud-Sandbox.git](https://github.com/ioos/Cloud-Sandbox.git)
```

The Sandbox environment provides built-in shell aliases to quickly navigate between your primary directories:
* `cds` — Changes directory to `/save/$USER` (model code, build scripts, and `Cloud-Sandbox` repo).
* `cdc` — Changes directory to `/com/$USER` (shared forcing files and model outputs).

---

## Next Steps

* **Working in Nebari:** Proceed to the **[Nebari User Guide](./NEBARI_USER_GUIDE.md)** to select an instance type, configure Conda Store environments, or use GPUs.
* **Understanding Storage & Data Transfers:** Review **[Storage & Data Management](./STORAGE_AND_DATA_MANAGEMENT.md)** before staging large datasets or model outputs.
* **Running Models:** Follow the **[Cloudflow Model Execution Instructions](./CLOUDFLOW_MODEL_EXECUTION_INSTRUCTIONS.md)** or **[Model Porting & Workflow Architecture](./MODEL_PORTING_AND_WORKFLOW.md)** to launch an on-demand compute cluster.



