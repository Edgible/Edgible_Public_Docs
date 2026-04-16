# Windows OS example: Evaluate Edgible self-hosting on disposable AWS EC2

This guide is **Windows Server on EC2 only**. It is for people who want to **assess Edgible’s self-hosting approach without installing trial software on their everyday PC**. You launch a **temporary Windows VM in AWS**, install **Docker for Windows containers**, run a **small proof container**, then use the same **six numbered steps** as **[Local-MacOS.md](Local-MacOS.md)** and **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)** (**Dashboard** + **CLI**), and **terminate** the instance when finished.

We do **not** use **WSL** here. **WSL 2** needs nested virtualization that **most small EC2 instance types do not provide**, and **WSL 1 + Docker** is fragile for real use. On cloud Windows, the dependable path is **Docker CE (Moby) on the host** with **Windows container images** from **Microsoft Container Registry**—the same class of stack as production Windows container hosts.

**Linux** images (for example **`nginx:alpine`**) belong on a **Linux** EC2 instance or on a physical Windows machine with **WSL 2** or **Docker Desktop**; they are **out of scope** for this document. For an **Ubuntu** EC2 sandbox (**SSH**, Docker, nginx, CLI), see **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)**.

To evaluate **Docker + Linux containers** entirely **on a Mac** (no Windows EC2), see **[Local-MacOS.md](Local-MacOS.md)**.

## How this guide is structured (repeatable framework)

| Step | What you do |
|------|----------------|
| **1.** | **Docker** and a **sample web workload** in a container (browser check) |
| **2.** | **Node.js** and **`@edgible-team/cli`** |
| **3.** | **Edgible account** in the browser (**Dashboard**), then **CLI authentication** |
| **4.** | **Install and run the Edgible agent** (CLI; **Dashboard** shows your device) |
| **5.** | **Register the sample container as an Edgible app** (**CLI** + **Dashboard**) |
| **6.** | **Teardown** (Edgible app / agent / CLI; stop/remove container; **terminate EC2**) |

Each step ends with **Verify / test**.

## Why EC2 for a sandbox

- **Isolation:** Workloads run in AWS, not on your home or work PC.
- **Low risk:** Use the **AWS console**, **RDP** (built into Windows; **Microsoft Remote Desktop** / **Windows App** on macOS), and optional browser checks.
- **Clean exit:** **Terminate** the instance; release **Elastic IPs**; remove unused **security groups** and volumes.

## Prerequisites (AWS EC2 — Windows Server)

### Suggested AMI and instance size

| Goal | Suggested starting point | Notes |
|------|---------------------------|--------|
| **IIS in Docker (web page in browser)** | **Windows Server 2025** or **2022**, License Included AMI | Prefer **`t3.small`** or larger and **≥ 30 GiB** root volume for the **Server Core + IIS** image; **`t3.micro`** often **runs out of RAM or disk** during `docker pull`. Match the **container tag** to your **Server LTSC** (see below). |
| **Edgible CLI + fuller self-hosting** | Scale instance type and disk per **official Edgible documentation** | |

**Container / host version:** Windows containers must match the **host OS build** family. Use **`windowsservercore-ltsc2025`** on **Windows Server 2025** and **`windowsservercore-ltsc2022`** on **Windows Server 2022**. See **[IIS Docker image](https://github.com/microsoft/iis-docker)** on GitHub if tags change.

**Connect from a Mac:** Install **Microsoft Remote Desktop** (or **Windows App**) from the App Store; add a PC with the instance **public IP**; user **`Administrator`**; password from EC2 **Get Windows password** (`.pem` key).

### Key pair (`.pem`) for RDP (Administrator password)

**Create a key pair before you launch the instance.** Windows Server on EC2 does not give you a default **Administrator** password in plain text; you decrypt it with the **private key** (**.pem**) that matches the **key pair** you selected at launch.

1. In the AWS console: **EC2** → **Key Pairs** → **Create key pair**.
2. Choose a **name** (for example **`edgible-windows-sandbox`**), key type **RSA** or **Ed25519**, and private key format **`.pem`**.
3. **Create key pair** and **download** the **`.pem`** file immediately. You **cannot** download the same private key again; if you lose it, use **EC2 Serial Console** / **replace the instance** / other recovery paths described in AWS docs—**Get Windows password** will not work without the **`.pem`**.
4. Store the file **only** on your own machine; do **not** commit it to git or leave it on untrusted shared storage.
5. On **macOS** or **Linux**, if you use the key file from the browser when decrypting the password, keep permissions tight: **`chmod 400 /path/to/your-key.pem`**.
6. When you **launch** the instance, under **Key pair (login)**, select this key pair.

### Connect to the instance

1. EC2 console: **Get Windows password** with your **`.pem`** key pair file.
2. **RDP** to the **public IP** or DNS name (**TCP 3389** from **your IP** in the security group).

Use **PowerShell as Administrator** for **steps 1–6** below (elevation matters on Windows; there is no **`sudo`**).

### Security group (inbound)

- **RDP:** TCP **3389** from **your IP** only.
- **HTTP:** TCP **80** from **your IP** (needed to load the IIS demo page in a browser).
- **HTTPS:** TCP **443** only if you add TLS later (egress for **`edgible auth login`** is usually allowed by default).

---

## 1. Docker (and IIS sample container)

Do **not** use **DockerMsftProvider** / `Install-Package -Name docker` (retired feed; installs fail).

### Install Docker CE (Windows containers)

1. **Elevated PowerShell:**

   ```powershell
   Invoke-WebRequest -UseBasicParsing "https://raw.githubusercontent.com/microsoft/Windows-Containers/Main/helpful_tools/Install-DockerCE/install-docker-ce.ps1" -OutFile install-docker-ce.ps1
   .\install-docker-ce.ps1
   ```

2. **Restart** if the script or Windows requests it; sign in again over RDP.

3. Read **[Set up your environment](https://learn.microsoft.com/en-us/virtualization/windowscontainers/quick-start/set-up-environment)** and **[Run your first Windows container](https://learn.microsoft.com/en-us/virtualization/windowscontainers/quick-start/run-your-first-container)** for image **tags** that match your **host OS / LTSC** (wrong tag = pull/run errors).

```powershell
docker version
```

### Run the IIS sample container

1. **Allow TCP 80 on Windows** (elevated PowerShell on the host):

   ```powershell
   New-NetFirewallRule -DisplayName "Edgible demo HTTP 80" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 80
   ```

2. **Run IIS** — use the line that matches your **Server** version (pull + first start can take **10+ minutes**; do not interrupt):

   **Windows Server 2025:**

   ```powershell
   docker run -d --name edgible-demo-iis -p 80:80 mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2025
   ```

   **Windows Server 2022:**

   ```powershell
   docker run -d --name edgible-demo-iis -p 80:80 mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022
   ```

3. When **`docker ps`** shows the container **running**, open **`http://<instance-public-ip>/`** in a browser (security group **TCP 80** from your IP). You should see the **default IIS** page.

**Logs (optional):** **`docker logs edgible-demo-iis`**.

**Important:** **Keep `edgible-demo-iis` running** through steps **4** and **5**. Do not **`docker stop`** / **`docker rm`** until **step 6** (unless you intentionally recreate it).

### Optional: tiny CLI check (Nano Server)

If you only want a **small** image to verify Docker before pulling IIS:

```powershell
docker run --rm mcr.microsoft.com/windows/nanoserver:ltsc2022 hostname
```

Use **`ltsc2025`** for **nanoserver** on Windows Server **2025** if that tag exists on MCR.

### Verify / test (step 1)

- [ ] **`docker version`** succeeds.
- [ ] Browser shows the **default IIS** page at **`http://<instance-public-ip>/`**.

If **`docker run` fails** with a version mismatch error, the image **tag** does not match your host **LTSC**; switch to the other line above or see **[Run your first Windows container](https://learn.microsoft.com/en-us/virtualization/windowscontainers/quick-start/run-your-first-container)**.

---

## 2. Node.js and Edgible CLI

The published package is **`@edgible-team/cli`** (command **`edgible`**). You need **Node.js 16+** and **npm** first.

### Install Node.js LTS (64-bit)

**Option A — winget** (Windows Server 2022 / 2025 with **App Installer** / winget available):

```powershell
winget install OpenJS.NodeJS.LTS --source winget --accept-package-agreements --accept-source-agreements
```

Use **`--source winget`** so installs come from the **winget** community index. If you omit it, **winget** may query the **Microsoft Store** source first; on some EC2 images that step fails with certificate errors (**`0x8a15005e`**), even though **Node.js (LTS)** is available from **winget**.

The installer updates **`PATH`**, but the **same** PowerShell window usually still has the old environment, so **`npm` is not recognized** until you start a fresh shell. **Easiest:** close PowerShell and open a **new** Administrator PowerShell, then run **Confirm** below and **`npm install -g @edgible-team/cli`**.

**Option B — MSI installer**

1. In the server’s browser (**Edge**), open **[https://nodejs.org/](https://nodejs.org/)** and download the **Windows Installer (.msi), 64-bit**, **LTS** build.
2. Run the MSI, accept defaults, and ensure **“Add to PATH”** stays enabled.
3. Open a **new** PowerShell window.

```powershell
node --version
npm --version
```

### Unzip on `PATH` (for local agent install)

Windows Server does not put **`unzip`** on **`PATH`** by default. The Edgible CLI shells out to **`unzip`** when it extracts the downloaded agent **`.zip`**, so **`edgible agent install`** can fail with **`Failed to install agent files`** / **`Command failed: unzip`** until **`unzip`** resolves.

**Install Git for Windows** (ships **`unzip.exe`** under **`usr\bin`**):

```powershell
winget install Git.Git --source winget --accept-package-agreements --accept-source-agreements
```

```powershell
& "C:\Program Files\Git\usr\bin\unzip.exe" -v
```

If **`unzip -v`** still fails but the path above works, **`usr\bin`** is not on **`PATH`**. Append it to the **machine** environment (Administrator PowerShell; adjust the folder if Git is installed elsewhere):

```powershell
$gitUsrBin = "C:\Program Files\Git\usr\bin"
$machinePath = [Environment]::GetEnvironmentVariable("Path", "Machine")
if ($machinePath -notlike "*${gitUsrBin}*") {
  [Environment]::SetEnvironmentVariable("Path", "$machinePath;$gitUsrBin", "Machine")
}
```

Open a **new** PowerShell window (or merge **Machine** + **User** **`Path`** into **`$env:Path`** for the current session), then **`unzip -v`** should work.

### Install the Edgible CLI globally

```powershell
npm install -g @edgible-team/cli
```

If PowerShell blocks scripts when you run **`edgible`**, set execution policy once (Administrator):

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
```

Then try again in a new window.

### Verify / test (step 2)

```powershell
edgible --version
edgible --help
```

- [ ] **`node --version`** shows **16+**.
- [ ] **`edgible --version`** prints a version string.

---

## 3. Edgible sign-up and `edgible auth login`

### 3.1 Browser: create or open your account (Dashboard)

On your **local machine** (Mac or PC with a browser—you can use RDP’s clipboard or type carefully on the server’s **Edge** if you prefer):

1. Open **[https://www.edgible.com](https://www.edgible.com)** (or **[https://www.edgible.com/signup](https://www.edgible.com/signup)**).
2. Click **Login** (typically **top right** on the public site).
3. Choose **Sign up here** (or equivalent) for a **new** account, or **sign in** if you already have one.
4. Land on the **Dashboard**. Keep it open for **steps 4** and **5** to compare with the **Windows Server** host where the CLI runs.

**Verify / test (3.1)**

- [ ] You see the **Dashboard** while logged in.

### 3.2 CLI on Windows Server: authenticate

Interactive login (needs outbound **HTTPS** to Edgible):

```powershell
edgible auth login
```

**Verify / test (3.2)**

- [ ] **`edgible auth login`** completes without errors.

Non-interactive and other auth flows: **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** (*First-time setup*, *Authentication*).

---

## 4. Start the Edgible agent

Run **`edgible agent install`** in **Administrator** PowerShell—the same elevated session you use for Docker when possible.

```powershell
edgible agent install
```

**Dashboard:** Refresh the **Dashboard** and look for this **Windows Server** instance as a **device**.

```powershell
edgible agent status
```

Use **`--watch`** for live updates (**Ctrl+C** to stop).

**Follow agent logs** (**`-f`** = **`--follow`**; **Ctrl+C** to stop):

```powershell
edgible agent logs -f
```

Use **`edgible agent logs -n 200`** for a snapshot; **`edgible agent logs --help`** lists filters.

### Verify / test (step 4)

- [ ] **`edgible agent status`** shows a healthy agent.
- [ ] **Dashboard** reflects this **device**.

---

## 5. Create an Edgible application (from the running container)

With **`edgible-demo-iis` still running** (host port **80** mapped as in **step 1**), after **steps 3–4**:

```powershell
edgible app create existing
```

Choose the **IIS** container workload and **port 80** when prompted (see **`edgible application create existing --help`**). **`edgible app`** is an alias for **`edgible application`**.

**Dashboard:** Confirm the new **application** appears and matches **`edgible app list`** / **`edgible app status`**.

```powershell
edgible app list
edgible app status
```

**`edgible app status`** is an alias for **`edgible app get`**. With multiple apps, use **`--app-id`** / **`-i`**.

If you hit **`Application handler not found`** / **`Handler not found`**, see the troubleshooting note in **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)** (same class of issue).

### Verify / test (step 5)

- [ ] **`edgible app ls`** (same command as **`edgible app list`**) lists your demo app. Note the **Serving IP**, **Port**, **Protocol**, and the **URL** line in the output (or run **`edgible app ls --json`** if you need machine-readable fields).
- [ ] **Dashboard** shows the **application** consistently with the CLI.
- [ ] **Internet (off-LAN):** On a **device that is not on the same LAN as the machine you use for RDP** (for example a **smartphone on cellular/mobile data** with **Wi‑Fi off** or **not** on your home/office Wi‑Fi), open a browser and go to the **URL** from **`edgible app ls`**, or **`http://` / `https://`** + **Serving IP** + **`:`** + **Port** as printed. You should see the same **IIS** default page as when you opened **`http://<instance-public-ip>/`** in step **1**, proving Edgible is **hosting the workload on the internet**. If it fails at first, wait for routing to propagate, check **`edgible app status`**, and retry.

---

## 6. Teardown (Edgible-related; EC2 cleanup)

### 6.1 Application (if created in step 5)

```powershell
edgible app delete
```

The CLI shows a **pick list** of applications; choose the one you created in step **5**. See **`edgible app delete --help`** for non-interactive flags (for example **`--app-id`**) if you need scripting.

### 6.2 Agent

```powershell
edgible agent uninstall --remove-files
```

Confirm flags in **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)**.

### 6.3 CLI session and package

```powershell
edgible auth logout
npm uninstall -g @edgible-team/cli
```

### 6.4 Demo container

```powershell
docker stop edgible-demo-iis
docker rm edgible-demo-iis
```

### 6.5 Optional: remove Docker CE

Only if you want the instance clean of Docker before terminate—follow Microsoft / Docker docs for your install path; many sandboxes simply **terminate** the instance.

### 6.6 EC2 and AWS cleanup

1. **Terminate** the EC2 instance (or stop if you will reuse it).
2. **Release** any **Elastic IP**.
3. Review **EBS** volumes and snapshots.
4. Remove unused **security group** rules.

**Dashboard:** Refresh after teardown; rows should update as Edgible’s backend syncs.

### Verify / test (step 6)

- [ ] Instance **terminated** (or stopped) and billing risk understood.
- [ ] Demo container / agent / CLI removed **if** you intended full cleanup.

That gives you a **Windows OS–specific**, WSL-free path through the same **six steps** as **[Local-MacOS.md](Local-MacOS.md)** and **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)**.
