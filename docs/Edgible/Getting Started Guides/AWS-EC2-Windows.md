# Windows OS example: Evaluate Edgible self-hosting on disposable AWS EC2

This guide is **Windows Server on EC2 only**. It is for people who want to **assess Edgible’s self-hosting approach without installing trial software on their everyday PC**. You launch a **temporary Windows VM in AWS**, install **Docker for Windows containers**, run a **small proof container**, then **terminate** the instance when finished.

We do **not** use **WSL** here. **WSL 2** needs nested virtualization that **most small EC2 instance types do not provide**, and **WSL 1 + Docker** is fragile for real use. On cloud Windows, the dependable path is **Docker CE (Moby) on the host** with **Windows container images** from **Microsoft Container Registry**—the same class of stack as production Windows container hosts.

**Linux** images (for example **`nginx:alpine`**) belong on a **Linux** EC2 instance or on a physical Windows machine with **WSL 2** or **Docker Desktop**; they are **out of scope** for this document. For an **Ubuntu** EC2 sandbox (**SSH**, Docker, nginx, CLI), see **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)**.

To evaluate **Docker + Linux containers** entirely **on a Mac** (no Windows EC2), see **[Local-MacOS.md](Local-MacOS.md)**.

## Why EC2 for a sandbox

- **Isolation:** Workloads run in AWS, not on your home or work PC.
- **Low risk:** Use the **AWS console**, **RDP** (built into Windows; **Microsoft Remote Desktop** / **Windows App** on macOS), and optional browser checks.
- **Clean exit:** **Terminate** the instance; release **Elastic IPs**; remove unused **security groups** and volumes.

## What you will run

1. **Microsoft’s Docker CE install script**, then the **official IIS Windows container** so you can open **`http://<public-ip>/`** in a browser (default IIS welcome page). Image pulls are **large** (several GB); first start can take **many minutes**.
2. **Node.js LTS** and the **Edgible CLI** (`@edgible-team/cli`) on the same Windows Server instance (see below).
3. **Optional next steps:** **`edgible auth login`**, then **`edgible agent install`** (Administrator PowerShell); apps, stacks, etc.—see **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)**.

## Suggested AMI and instance size

| Goal | Suggested starting point | Notes |
|------|---------------------------|--------|
| **IIS in Docker (web page in browser)** | **Windows Server 2025** or **2022**, License Included AMI | Prefer **`t3.small`** or larger and **≥ 30 GiB** root volume for the **Server Core + IIS** image; **`t3.micro`** often **runs out of RAM or disk** during `docker pull`. Match the **container tag** to your **Server LTSC** (see below). |
| **Edgible CLI + fuller self-hosting** | Scale instance type and disk per **official Edgible documentation** | |

**Container / host version:** Windows containers must match the **host OS build** family. Use **`windowsservercore-ltsc2025`** on **Windows Server 2025** and **`windowsservercore-ltsc2022`** on **Windows Server 2022**. See **[IIS Docker image](https://github.com/microsoft/iis-docker)** on GitHub if tags change.

**Connect from a Mac:** Install **Microsoft Remote Desktop** (or **Windows App**) from the App Store; add a PC with the instance **public IP**; user **`Administrator`**; password from EC2 **Get Windows password** (`.pem` key).

## Key pair (`.pem`) for RDP (Administrator password)

**Create a key pair before you launch the instance.** Windows Server on EC2 does not give you a default **Administrator** password in plain text; you decrypt it with the **private key** (**.pem**) that matches the **key pair** you selected at launch.

1. In the AWS console: **EC2** → **Key Pairs** → **Create key pair**.
2. Choose a **name** (for example **`edgible-windows-sandbox`**), key type **RSA** or **Ed25519**, and private key format **`.pem`**.
3. **Create key pair** and **download** the **`.pem`** file immediately. You **cannot** download the same private key again; if you lose it, use **EC2 Serial Console** / **replace the instance** / other recovery paths described in AWS docs—**Get Windows password** will not work without the **`.pem`**.
4. Store the file **only** on your own machine; do **not** commit it to git or leave it on untrusted shared storage.
5. On **macOS** or **Linux**, if you use the key file from the browser when decrypting the password, keep permissions tight: **`chmod 400 /path/to/your-key.pem`**.
6. When you **launch** the instance, under **Key pair (login)**, select this key pair.

## Connect to the instance

1. EC2 console: **Get Windows password** with your **`.pem`** key pair file.
2. **RDP** to the **public IP** or DNS name (**TCP 3389** from **your IP** in the security group).

Use **PowerShell as Administrator** for Docker steps below.

## Security group (inbound)

- **RDP:** TCP **3389** from **your IP** only.
- **HTTP:** TCP **80** from **your IP** (needed to load the IIS demo page in a browser).
- **HTTPS:** TCP **443** only if you add TLS later.
- **Additional ports:** Only what Edgible or your app docs require.

## Install Docker CE (Windows containers)

Do **not** use **DockerMsftProvider** / `Install-Package -Name docker` (retired feed; installs fail).

1. **Elevated PowerShell:**

   ```powershell
   Invoke-WebRequest -UseBasicParsing "https://raw.githubusercontent.com/microsoft/Windows-Containers/Main/helpful_tools/Install-DockerCE/install-docker-ce.ps1" -OutFile install-docker-ce.ps1
   .\install-docker-ce.ps1
   ```

2. **Restart** if the script or Windows requests it; sign in again over RDP.

3. Verify:

   ```powershell
   docker version
   ```

4. Read **[Set up your environment](https://learn.microsoft.com/en-us/virtualization/windowscontainers/quick-start/set-up-environment)** and **[Run your first Windows container](https://learn.microsoft.com/en-us/virtualization/windowscontainers/quick-start/run-your-first-container)** for image **tags** that match your **host OS / LTSC** (wrong tag = pull/run errors).

## Sample web app: IIS in a Windows container

Microsoft publishes **IIS on Windows Server Core** as a ready-made image: default site answers on **port 80** inside the container. Map it to the host so your browser can reach it.

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

3. When **`docker ps`** shows the container **running**, open **`http://<instance-public-ip>/`** in a browser (same **security group** rule for **TCP 80** from your IP). You should see the **default IIS** page.

4. **Logs / status** (optional):

   ```powershell
   docker logs edgible-demo-iis
   ```

5. **Stop and remove** when finished:

   ```powershell
   docker stop edgible-demo-iis
   docker rm edgible-demo-iis
   ```

If **`docker run` fails** with a version mismatch error, the image **tag** does not match your host **LTSC**; switch to the other line above or see **[Run your first Windows container](https://learn.microsoft.com/en-us/virtualization/windowscontainers/quick-start/run-your-first-container)**.

### Optional: tiny CLI check (Nano Server)

If you only want a **small** image to verify Docker before pulling IIS:

```powershell
docker run --rm mcr.microsoft.com/windows/nanoserver:ltsc2022 hostname
```

Use **`ltsc2025`** for **nanoserver** on Windows Server **2025** if that tag exists on MCR; otherwise rely on the **IIS** step above for a browser-visible demo.

## Install Edgible CLI on Windows Server

The published package is **`@edgible-team/cli`** (command name **`edgible`**). You need **Node.js 16+** and **npm** first. Run these in **PowerShell** (Administrator is fine; required for some installs).

### 1. Install Node.js LTS (64-bit)

**Option A — winget** (Windows Server 2022 / 2025 with **App Installer** / winget available):

```powershell
winget install OpenJS.NodeJS.LTS --source winget --accept-package-agreements --accept-source-agreements
```

Use **`--source winget`** so installs come from the **winget** community index. If you omit it, **winget** may query the **Microsoft Store** source first; on some EC2 images that step fails with certificate errors (**`0x8a15005e`**, “server certificate did not match”), even though **Node.js (LTS)** is available from **winget**.

The installer updates **`PATH`**, but the **same** PowerShell window usually still has the old environment, so **`npm` is not recognized** until you start a fresh shell. **Easiest:** close PowerShell and open a **new** Administrator PowerShell, then run **Confirm** below and **`npm install -g @edgible-team/cli`**. (Signing out and back in also works if you prefer.)

**Option B — MSI installer**

1. In the server’s browser (**Edge**), open **[https://nodejs.org/](https://nodejs.org/)** and download the **Windows Installer (.msi), 64-bit**, **LTS** build.
2. Run the MSI, accept defaults, and ensure **“Add to PATH”** stays enabled.
3. Open a **new** PowerShell window.

Confirm:

```powershell
node --version
npm --version
```

### 2. Unzip on `PATH` (for local agent install)

Windows Server does not put **`unzip`** on **`PATH`** by default. The Edgible CLI shells out to **`unzip`** when it extracts the downloaded agent **`.zip`**, so **`edgible agent`** (or similar) can fail with **`Failed to install agent files`** / **`Command failed: unzip`** until **`unzip`** resolves in PowerShell.

**Install Git for Windows** (ships **`unzip.exe`** under **`usr\bin`**):

```powershell
winget install Git.Git --source winget --accept-package-agreements --accept-source-agreements
```

Confirm the binary exists:

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

Open a **new** PowerShell window (or refresh this session: merge **Machine** + **User** **`Path`** into **`$env:Path`** as in the Node.js step), then **`unzip -v`** should work. You only need this if you install or run the agent from the CLI on this machine; skip if you are only checking **`edgible --version`**.

### 3. Install the Edgible CLI globally

```powershell
npm install -g @edgible-team/cli
```

If PowerShell blocks scripts when you run **`edgible`**, set execution policy once (Administrator):

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
```

Then try again in a new window.

### 4. Verify

```powershell
edgible --version
edgible --help
```

### 5. First-time sign-in (optional)

Interactive login (needs outbound **HTTPS** to Edgible; default EC2 security group allows this):

```powershell
edgible auth login
```

To **install the Edgible agent** on this server (Windows service, **`ProgramData`**, etc.), run **`edgible agent install`** in the same **Administrator** PowerShell session—Windows has no **`sudo`**; elevation is what matters.

To **see whether the agent and daemon are healthy** after install:

```powershell
edgible agent status
```

Use **`--watch`** if you want live updates until you press **Ctrl+C**.

**Follow agent logs** (**`-f`** = **`--follow`**; **Ctrl+C** to stop):

```powershell
edgible agent logs -f
```

Administrator session is usually enough on Windows after **`edgible agent install`**. Use **`edgible agent logs -n 200`** for a snapshot; **`edgible agent logs --help`** lists filters.

Non-interactive and other auth flows: **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** (sections *First-time setup* and *Authentication*).

To **list applications** or **inspect one** (after you create any on this or another device): **`edgible app list`** and **`edgible app status`** (alias of **`edgible app get`**; use **`--app-id`** / **`-i`** when you have several).

Further commands (`application`, `gateway`, `stack`, `ai`, …) are documented in the same guide.

## When you are done

1. **Terminate** the EC2 instance (or stop if you will reuse it).
2. **Release** any **Elastic IP**.
3. Review **EBS** volumes and snapshots.
4. Remove unused **security group** rules.

That gives you a **Windows OS–specific**, WSL-free way to validate Docker on EC2 and continue evaluating Edgible self-hosting from a disposable cloud VM.
