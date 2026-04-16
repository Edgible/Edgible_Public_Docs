# Local example: Evaluate Edgible on your Mac (macOS)

This guide is **macOS on your own Mac only** (MacBook, Mac mini, Mac Studio, etc.). You use **Docker** with a **small Linux container** you can open in a browser, then optional **Node.js** and the **Edgible CLI** (`@edgible-team/cli`). It mirrors the flow in **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)** but runs **on your machine** instead of a disposable cloud VM.

For **isolation** and **no local installs**, use a cloud sandbox instead: **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)** (Linux / SSH) or **[AWS-EC2-Windows.md](AWS-EC2-Windows.md)** (Windows Server / RDP).

## Why use your Mac

- **Fast iteration:** No EC2 key pairs, security groups, or billing—just Docker and a terminal.
- **Familiar tooling:** Same **Linux containers** as many production paths (**`nginx:alpine`** here).
- **Tradeoffs:** Workloads share your Mac’s CPU, RAM, and disk; mistakes affect **your** user account unless you use VMs separately. This is a **dev / evaluation** path, not a substitute for hardened production hosting.

## What you will run

1. **Docker Desktop** (or another **Docker Engine** backend for Mac you already use), then **`nginx:alpine`** mapped to a host port so **`http://127.0.0.1:<port>/`** works in a browser.
2. **Node.js LTS** and **`npm install -g @edgible-team/cli`**.
3. **Optional:** **`edgible auth login`**, then **`sudo edgible agent install`**, then **`edgible app create existing`** on the **still-running** nginx container (**do not** stop/remove it before that step), then **`edgible app status`** (or **`edgible app list`**). Full command reference: **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)**.

## Before you start

| Topic | Notes |
|--------|--------|
| **macOS version** | Use a **currently supported** macOS release. Older macOS may not run current Docker Desktop builds. |
| **Apple Silicon (M1/M2/M3/…)** vs **Intel** | **`nginx:alpine`** is multi-arch; no extra steps for this demo. If you pull **x86-only** images on Apple Silicon, you may need **Rosetta** / **`platform: linux/amd64`**—not required here. |
| **Administrator access** | Installing **Docker Desktop** and optional **Edgible agent** system components may prompt for an **administrator password**. |
| **VPN / corporate proxy** | Docker pulls and **`edgible auth login`** need working **HTTPS** outbound; adjust per your org’s policy. |

## Install Docker (macOS)

Follow the vendor’s current docs if these steps drift.

### Option A — Docker Desktop (recommended for this guide)

1. Download **Docker Desktop for Mac** from **[Install Docker Desktop on Mac](https://docs.docker.com/desktop/install/mac-install/)** (Apple Silicon vs Intel installers differ).
2. Open the **`.dmg`**, drag **Docker** to **Applications**, launch **Docker**, and complete the first-run wizard.
3. Wait until the whale icon shows **Docker is running** (menu bar).

**Homebrew (optional):**

```bash
brew install --cask docker
```

Then start **Docker** from **Applications** (or Spotlight) once; the cask does not always auto-start the app.

### Option B — Other engines (advanced)

If you already run **Colima**, **Rancher Desktop**, or **OrbStack** with a **Docker CLI** that talks to a Linux VM, you can use that instead of Docker Desktop—ensure **`docker version`** works from **Terminal** before continuing.

Verify:

```bash
docker version
docker run --rm hello-world
```

If **`Cannot connect to the Docker daemon`**, the Docker engine is not running—start **Docker Desktop** (or your Colima/Rancher/OrbStack stack) and retry.

## Sample web app: nginx in a Linux container

**1. Pick a host port**

- **Port 80** often works with **Docker Desktop** as **`http://127.0.0.1/`**.
- If **80** is in use or your browser shows a conflict, use **8080** (or any free high port):

```bash
docker run -d --name edgible-demo-nginx -p 8080:80 nginx:alpine
```

For **80** explicitly:

```bash
docker run -d --name edgible-demo-nginx -p 80:80 nginx:alpine
```

**2. Open in a browser**

- **`http://127.0.0.1/`** if you mapped **`-p 80:80`**.
- **`http://127.0.0.1:8080/`** if you mapped **`-p 8080:80`**.

**3. Logs** (optional):

```bash
docker logs edgible-demo-nginx
```

**Keep this container running** if you continue with **`edgible app create existing`** below: that flow expects a **live** workload. **Do not** **`docker stop`** / **`docker rm`** **`edgible-demo-nginx`** until you are finished with Edgible for this session.

**4. Stop and remove nginx** when you are done:

```bash
docker stop edgible-demo-nginx
docker rm edgible-demo-nginx
```

## macOS firewall (optional)

If **System Settings** → **Network** → **Firewall** blocks local services, allow **Docker** / **com.docker.backend** when prompted, or temporarily test with the firewall off **only** on a trusted home network. Prefer **narrow rules** over turning security off globally.

## Install Node.js LTS

The Edgible CLI needs **Node.js 16+** and **npm**.

Pick **one** approach:

### A — **nvm** (clean global npm without `sudo`)

Use the current install command from **[nvm — Installing and updating](https://github.com/nvm-sh/nvm#installing-and-updating)** (one-liner **`curl … | bash`**), then restart the shell or **`source ~/.zshrc`** (or **`~/.bashrc`**):

```bash
nvm install --lts
nvm use --lts
node --version
npm --version
```

### B — **Homebrew**

```bash
brew install node
node --version
npm --version
```

### C — **Official installer**

Download the **LTS** **`.pkg`** from **[https://nodejs.org](https://nodejs.org)** and install.

## Install the Edgible CLI globally

If **`npm install -g`** fails with **permission denied** on stock **Homebrew Node**, either fix npm’s global prefix (see npm docs) or install under your user via **nvm** (recommended above).

```bash
npm install -g @edgible-team/cli
```

If you deliberately use **`sudo npm install -g …`**, understand it mixes **root-owned** files into global npm; **nvm** avoids that.

Verify:

```bash
edgible --version
edgible --help
```

## First-time sign-in (optional)

```bash
edgible auth login
```

More auth options: **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** (*First-time setup*, *Authentication*).

## Install the Edgible agent on this Mac (optional)

After **`edgible auth login`**, the installer may need elevated rights to register **LaunchDaemon**s and system paths:

```bash
sudo edgible agent install
```

Check status (use **`sudo`** consistently if the installer stored root-owned config, mirroring the **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)** flow):

```bash
sudo edgible agent status
sudo edgible agent logs -n 200
```

See the **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** for agent details and log options.

## Register the running nginx as an Edgible application (optional)

With **`edgible-demo-nginx` still up** and the same port you chose (**80** or **8080**), after **sign-in** and **agent install**:

```bash
edgible app create existing
```

Choose the **nginx** workload and the **container port** exposed to clients (**80** inside the container). If you published the host as **8080→80**, follow CLI prompts so the app’s public URL matches how you reach it (see **`edgible application create existing --help`** and the **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)**).

Inspect apps:

```bash
edgible app list
edgible app status
```

**Troubleshooting — agent log: `Application handler not found` / `Workload status check failed: Handler not found`**

Same class of issue as on **Ubuntu EC2**: **`edgible app create existing`** registers the app with the **API** but local wiring can lag what the agent expects. Use **`sudo edgible agent logs -f`**, **`edgible device application-health`**, and org support guidance; see the note in **[AWS-EC2-Ubuntu.md](AWS-EC2-Ubuntu.md)** for the longer explanation.

## When you are done

1. **Optional:** **`docker stop edgible-demo-nginx`** and **`docker rm edgible-demo-nginx`**.
2. **Optional:** remove the global CLI (**`npm uninstall -g @edgible-team/cli`**) if you no longer want it.
3. **Quit Docker Desktop** (or stop Colima/Rancher/OrbStack) if you want Docker offloaded from memory.

That gives you a **local macOS** path to validate **Docker + nginx** and continue evaluating Edgible without AWS.
