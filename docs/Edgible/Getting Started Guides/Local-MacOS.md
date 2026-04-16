# Local example: Evaluate Edgible on your Mac (macOS)

This guide is **macOS on your own Mac only** (MacBook, Mac mini, Mac Studio, etc.). It follows the same **numbered steps** as **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)** and **[AWS-EC2-Windows.md](AWS-EC2-Windows.md)** so you can compare **local Mac**, **Linux EC2**, and **Windows EC2** side by side.

For **isolation** and **no local installs**, use a cloud sandbox: **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)** or **[AWS-EC2-Windows.md](AWS-EC2-Windows.md)**.

## Why use your Mac

- **Fast iteration:** No EC2 key pairs, security groups, or billing—just Docker and a terminal.
- **Familiar tooling:** Same **Linux containers** as many production paths (**`nginx:alpine`** here).
- **Tradeoffs:** Workloads share your Mac’s CPU, RAM, and disk; mistakes affect **your** user account unless you use VMs separately. This is a **dev / evaluation** path, not a substitute for hardened production hosting.

## How this guide is structured (repeatable framework)

| Step | What you do |
|------|----------------|
| **1.** | **Docker** and a **sample web workload** in a container (browser check) |
| **2.** | **Node.js** and **`@edgible-team/cli`** |
| **3.** | **Edgible account** in the browser (**Dashboard**), then **CLI authentication** |
| **4.** | **Install and run the Edgible agent** (CLI; **Dashboard** shows your device) |
| **5.** | **Register the sample container as an Edgible app** (**CLI** + **Dashboard**) |
| **6.** | **Teardown** (Edgible app / agent / CLI; optional Docker removal) |

Each step ends with **Verify / test**.

## Before you start (Mac-specific)

| Topic | Notes |
|--------|--------|
| **macOS version** | Use a **currently supported** macOS release. Older macOS may not run current Docker Desktop builds. |
| **Apple Silicon (M1/M2/M3/…)** vs **Intel** | **`nginx:alpine`** is multi-arch; no extra steps for this demo. |
| **Administrator access** | **Docker Desktop** and the **Edgible agent** may prompt for an **administrator password**. |
| **VPN / corporate proxy** | Docker pulls and **`edgible auth login`** need working **HTTPS** outbound. |

### macOS firewall (optional)

If **System Settings** → **Network** → **Firewall** blocks local services, allow **Docker** / **com.docker.backend** when prompted, or temporarily test with the firewall off **only** on a trusted home network.

---

## 1. Docker (and nginx sample)

Follow the vendor’s current docs if these drift.

### Install Docker on macOS

**Option A — Docker Desktop (recommended)**

1. Download **Docker Desktop for Mac** from **[Install Docker Desktop on Mac](https://docs.docker.com/desktop/install/mac-install/)** (Apple Silicon vs Intel installers differ).
2. Open the **`.dmg`**, drag **Docker** to **Applications**, launch **Docker**, and complete the first-run wizard.
3. Wait until the whale icon shows **Docker is running** (menu bar).

**Homebrew (optional):**

```bash
brew install --cask docker
```

Then start **Docker** from **Applications** (or Spotlight) once.

**Option B — Other engines (advanced)**

If you already run **Colima**, **Rancher Desktop**, or **OrbStack** with a **Docker CLI** that talks to a Linux VM, use that instead—ensure **`docker version`** works from **Terminal**.

### Run the sample container (`nginx:alpine`)

**Pick a host port**

- **Port 80** often works as **`http://127.0.0.1/`**.
- If **80** is in use, use **8080**:

```bash
docker run -d --name edgible-demo-nginx -p 8080:80 nginx:alpine
```

Or for **80**:

```bash
docker run -d --name edgible-demo-nginx -p 80:80 nginx:alpine
```

**Open in a browser:** **`http://127.0.0.1/`** (port **80**) or **`http://127.0.0.1:8080/`** (port **8080**).

**Logs (optional):** **`docker logs edgible-demo-nginx`**.

**Important:** **Keep `edgible-demo-nginx` running** through steps **4** and **5**. Do not **`docker stop`** / **`docker rm`** it until you begin **step 6** (or you intentionally recreate it).

### Verify / test (step 1)

- [ ] **`docker version`** and **`docker run --rm hello-world`** succeed (no “Cannot connect to the Docker daemon”).
- [ ] Browser shows **Welcome to nginx!** on the URL you chose (**127.0.0.1** with the right port).

---

## 2. Node.js and Edgible CLI

The Edgible CLI needs **Node.js 16+** and **npm**.

### Install Node.js LTS

Pick **one** approach:

**A — nvm** (clean global npm without `sudo`)

Use the current install command from **[nvm — Installing and updating](https://github.com/nvm-sh/nvm#installing-and-updating)** (one-liner **`curl … | bash`**), then restart the shell or **`source ~/.zshrc`** (or **`~/.bashrc`**):

```bash
nvm install --lts
nvm use --lts
node --version
npm --version
```

**B — Homebrew:** **`brew install node`**

**C — Official installer:** **[https://nodejs.org](https://nodejs.org)** LTS **`.pkg`**.

### Install the CLI globally

```bash
npm install -g @edgible-team/cli
```

If **`npm install -g`** fails with **permission denied**, fix npm’s global prefix (see npm docs) or use **nvm** (recommended). Avoid **`sudo npm install -g`** unless you accept **root-owned** global packages.

### Verify / test (step 2)

```bash
edgible --version
edgible --help
```

- [ ] **`node --version`** shows **16+**.
- [ ] **`edgible --version`** prints a version string.

---

## 3. Edgible sign-up and `edgible auth login`

You need an **Edgible account** in the browser (so you have a **Dashboard**), then you link the **CLI** on this Mac with **`edgible auth login`**.

### 3.1 Browser: create or open your account (Dashboard)

1. Open **[https://www.edgible.com](https://www.edgible.com)** in a browser (or **[https://www.edgible.com/signup](https://www.edgible.com/signup)** to go straight to sign-up).
2. Click **Login** (typically **top right** on the public site).
3. On the Edgible auth screen, choose **Sign up here** (or equivalent) if you need a **new** account, or **sign in** if you already have credentials.
4. After authentication you should land on the **Dashboard**—the browser home for your account, **devices**, and **applications**. Keep this tab handy for **steps 4** and **5**: you can confirm CLI actions (new device, app registration) here as well as in the terminal.

**Verify / test (3.1)**

- [ ] While logged in, you see the **Dashboard** (not only the marketing homepage).

### 3.2 CLI: authenticate this machine

```bash
edgible auth login
```

Complete the flow the CLI prints (often a browser/device code). Needs outbound **HTTPS**.

**Verify / test (3.2)**

- [ ] **`edgible auth login`** completes without errors.
- [ ] Optional: run a read-only command such as **`edgible app list`** (may be empty) to confirm the session.

More auth options: **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** (*First-time setup*, *Authentication*).

---

## 4. Start the Edgible agent

Install the **agent** so this Mac can register as a **device** and run Edgible workloads. Use **`sudo`** so the installer can place **LaunchDaemon**s and system paths:

```bash
sudo edgible agent install
```

**Dashboard:** After the agent registers, refresh the **Dashboard** and look for this **device** (hostname / labels depend on Edgible UI). If something looks wrong, fix the CLI first, then re-check the Dashboard—they should agree.

**Status and logs** (use **`sudo`** consistently if the installer used root-owned config, same pattern as **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)**):

```bash
sudo edgible agent status
sudo edgible agent logs -n 200
```

Add **`--watch`** on **`agent status`** for live updates (**Ctrl+C** to stop). For streaming logs: **`sudo edgible agent logs -f`**.

See **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** for **`agent install`**, log filters, and troubleshooting.

### Verify / test (step 4)

- [ ] **`sudo edgible agent status`** shows a healthy agent (per CLI output).
- [ ] **Dashboard** lists this machine as a **device** (or shows expected registration state—wording may vary by Edgible version).

---

## 5. Create an Edgible application (from the running container)

With **`edgible-demo-nginx` still running** and **steps 3–4** complete:

```bash
edgible app create existing
```

**`app`** is a short alias for **`application`**. The CLI prompts for an **application name**, **serving device** (this Mac, once registered), **workload** (choose **`edgible-demo-nginx`**), and **port** (**80** inside the container). If you mapped the host to **8080→80**, follow prompts and **`edgible application create existing --help`** so public access matches how you browse locally.

**Dashboard:** Open or refresh the **Dashboard** and find the new **application** alongside the **device**. You can cross-check **`edgible app list`** / **`edgible app status`** with what the UI shows.

```bash
edgible app list
edgible app status
```

**`edgible app status`** is an alias for **`edgible app get`**. With multiple apps, use **`--app-id`** / **`-i`** from **`edgible app list`**.

### Troubleshooting — `Application handler not found` / `Handler not found`

The **serving agent** may try to health-check an **`appId`** the **API** knows about before local handler wiring completes. Use **`sudo edgible agent logs -f`**, **`edgible device application-health --help`**, and the longer note in **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)** (same class of issue on Linux EC2).

### Verify / test (step 5)

- [ ] **`edgible app ls`** (same command as **`edgible app list`**) lists your demo app. Note the **Serving IP**, **Port**, **Protocol**, and the **URL** line in the output (or run **`edgible app ls --json`** if you need machine-readable fields).
- [ ] **Dashboard** shows the **application** tied to your account/device in a way that matches the CLI.
- [ ] **Internet (off-LAN):** On a **device that is not on your local network**—for example a **smartphone on cellular/mobile data** with **Wi‑Fi off** or **not** connected to your home Wi‑Fi—open a browser and go to the **URL** shown by **`edgible app ls`**, or build **`http://` or `https://`** + **Serving IP** + **`:`** + **Port** to match the CLI. You should see the same **nginx** welcome page as on your Mac, proving the app is **reachable from the internet**, not only on your LAN. If it does not load yet, wait briefly for Edgible routing to settle, confirm **`edgible app status`** shows **active**, and retry.

---

## 6. Teardown (Edgible-related; optional Docker)

Do these in a sensible order for your machine (stop workloads before removing the agent if your org recommends that).

### 6.1 Application (if you created one in step 5)

```bash
edgible app delete
```

The CLI shows a **pick list** of applications; choose the one you created in step **5**. See **`edgible app delete --help`** for non-interactive flags (for example **`--app-id`**) if you need scripting.

### 6.2 Agent

```bash
sudo edgible agent uninstall --remove-files
```

Confirm flags in **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** (`edgible agent uninstall`).

### 6.3 CLI session and package

```bash
edgible auth logout
npm uninstall -g @edgible-team/cli
```

### 6.4 Demo container

```bash
docker stop edgible-demo-nginx
docker rm edgible-demo-nginx
```

### 6.5 Optional: remove Docker entirely

If you installed **Docker Desktop** only for this evaluation: **Quit Docker**, then delete **Docker** from **Applications** (or **`brew uninstall --cask docker`**). Colima / OrbStack users should follow that product’s uninstall docs.

**Dashboard:** You can sign out or leave the account as-is; devices/apps removed from the host may still show until Edgible’s backend catches up—refresh the **Dashboard** after a short wait.

### Verify / test (step 6)

- [ ] No **`edgible-demo-nginx`** in **`docker ps -a`** (if you removed it).
- [ ] **`edgible`** not on **`PATH`** (if you uninstalled the CLI).
- [ ] **Dashboard** no longer shows the device/app as active, or matches your expectations after uninstall.

That gives you a **local macOS** path through the same **six steps** used in the EC2 guides.
