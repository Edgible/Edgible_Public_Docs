# Linux example: Evaluate Edgible self-hosting on disposable AWS EC2 (Ubuntu)

This guide is **Ubuntu LTS on EC2 only**. It mirrors **[AWS-EC2-Windows.md](AWS-EC2-Windows.md)** for people who prefer a **Linux** sandbox: **SSH**, **Docker Engine** with a **small Linux container** you can hit in a browser, then optional **Node.js** and the **Edgible CLI** (`@edgible-team/cli`). **Terminate** the instance when finished.

For **Windows Server**, **RDP**, and **Windows containers (IIS)**, use the Windows doc above.

For the **same nginx + Docker + CLI** idea **on your Mac** without EC2, see **[Local-MacOS.md](Local-MacOS.md)**.

## Why EC2 for a sandbox

- **Isolation:** Workloads run in AWS, not on your laptop.
- **Low cost / low risk:** **Ubuntu** + **Docker** + **`nginx:alpine`** fit comfortably on a **free-tier–eligible** instance type where your account still has that allowance; always confirm in **[AWS Free Tier](https://aws.amazon.com/free/)** and **Billing**.
- **Clean exit:** **Terminate** the instance; release **Elastic IPs**; remove unused **security groups** and volumes.

## What you will run

1. **Docker Engine** from Docker’s **apt** repository, then **`nginx:alpine`** mapped to **port 80** so **`http://<public-ip>/`** works in a browser.
2. **Node.js LTS** and **`npm install -g @edgible-team/cli`** on the same instance.
3. **Optional:** **`edgible auth login`**, then **`sudo edgible agent install`**, then **`edgible app create existing`** on the **still-running** nginx container (**do not** stop/remove it before that step), then **`edgible app status`** (or **`edgible app list`**) to inspect the app. Other commands: **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)**.

## Suggested AMI and instance size

| Goal | Suggested starting point | Notes |
|------|---------------------------|--------|
| **nginx in Docker (quick browser check)** | **Ubuntu Server 22.04 or 24.04 LTS** (Canonical AMI) | **`t3.micro`** is often enough for pull + run; see **root volume** below. |
| **Edgible CLI + agent / heavier self-hosting** | **`t3.small`** or larger, more disk | Follow main Edgible self-hosting docs for production-like sizing. |

**Root volume (EBS) for this guide:** For **Ubuntu + Docker + `nginx:alpine` only**, **20 GiB** is **enough** and a comfortable default (OS, Docker, one tiny image, logs, **`apt`/`npm`** headroom). **8–10 GiB** can work for the bare minimum demo but gets **tight** if you pull **larger** images, keep **Docker** cache, or run **`npm install -g`** with many dependencies—use **≥ 20 GiB** if you are unsure.

**AMI:** In the EC2 launch wizard, search **Ubuntu** and pick **64-bit (x86)** unless you intentionally use **Graviton (arm64)**—then use arm64 AMIs and matching Docker **`arch=`** in the apt repo line (see Docker’s Ubuntu install page).

**Default user:** Canonical AMIs use **`ubuntu`** (not `ec2-user`). Example: **`ssh -i your-key.pem ubuntu@<public-ip>`**.

## Key pair (`.pem`) for SSH login

**Create a key pair before you launch the instance** so you can log in remotely with **`ssh -i …`**.

1. In the AWS console: **EC2** → **Key Pairs** → **Create key pair**.
2. Choose a **name** (for example **`edgible-ubuntu-sandbox`**), key type **RSA** or **Ed25519**, and private key format **`.pem`** (not **`.ppk`** unless you will convert it for PuTTY yourself).
3. **Create key pair** and **download** the **`.pem`** file immediately. AWS does **not** let you download the same private key again later.
4. Store the file **only** on your own machine; do **not** commit it to git or leave it on shared storage you do not trust.
5. On **macOS** or **Linux**, restrict permissions so **OpenSSH** accepts the key:

   ```bash
   chmod 400 /path/to/your-key.pem
   ```

6. When you **launch** the instance, under **Key pair (login)**, select this key pair. If you launch without a key pair you control, you may **not** be able to SSH in at all (unless you later attach another access path such as **EC2 Instance Connect** with different requirements).

## Connect to the instance

1. Launch (or relaunch) the instance with that **key pair** selected; security group allows **TCP 22** from **your IP**.
2. From your Mac or PC:

   ```bash
   ssh -i /path/to/your-key.pem ubuntu@<instance-public-ip>
   ```

Use **`sudo`** where indicated below (Ubuntu’s **`ubuntu`** user has passwordless sudo).

## Security group (inbound)

- **SSH:** TCP **22** from **your IP** only.
- **HTTP:** TCP **80** from **your IP** (for the nginx demo in a browser).
- **HTTPS:** TCP **443** only if you add TLS or the Edgible docs require it.

## Install Docker Engine (Ubuntu)

Follow Docker’s current steps if these drift: **[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)**.

**Summary (Docker apt repo, non-interactive):**

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Run containers as **`ubuntu`** without typing `sudo docker` every time:

```bash
sudo usermod -aG docker ubuntu
```

**Log out and SSH back in** (or run **`newgrp docker`**) so the **`docker`** group applies.

Verify:

```bash
docker version
docker run --rm hello-world
```

If **`permission denied`** on **`/var/run/docker.sock`**, you skipped **`usermod`** or your shell does not have the **`docker`** group yet—run **`sudo usermod -aG docker ubuntu`**, then **log out and SSH in again** or **`newgrp docker`**. Until then, **`sudo docker …`** works as a temporary workaround.

## Sample web app: nginx in a Linux container

**1. Optional host firewall (UFW)** — only if you enabled **UFW**; many sandboxes rely on the **security group** alone:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw enable   # skip if you do not use UFW
```

**2. Run nginx** (small image; first pull is quick):

```bash
docker run -d --name edgible-demo-nginx -p 80:80 nginx:alpine
```

**3.** When **`docker ps`** shows the container **Up**, open **`http://<instance-public-ip>/`** in a browser (security group must allow **TCP 80** from your IP). You should see the **“Welcome to nginx!”** page.

**4. Logs** (optional):

```bash
docker logs edgible-demo-nginx
```

**Keep this container running** if you continue with **step 7** below (**`edgible app create existing`**): that flow expects a **live** workload on the host (the CLI can list running containers and bind the app to a port such as **80**). **Do not** run **`docker stop`** / **`docker rm`** on **`edgible-demo-nginx`** until you are finished with Edgible for this sandbox.

**5. Stop and remove nginx** — only when you are **done** with the demo (or you have recreated nginx on another port/name):

```bash
docker stop edgible-demo-nginx
docker rm edgible-demo-nginx
```

## Install Edgible CLI on Ubuntu

The published package is **`@edgible-team/cli`** (command **`edgible`**). You need **Node.js 16+** and **npm**.

### 1. Node.js LTS (recommended: NodeSource)

Ubuntu’s default **`apt install nodejs`** can be **too old** for the CLI. Use **[NodeSource](https://github.com/nodesource/distributions)** (check their README for the current setup script name), for example:

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs
```

Confirm:

```bash
node --version
npm --version
```

### 2. `unzip` (only if `edgible agent` install complains)

Most Ubuntu images already have **`unzip`**. If a local agent install fails on **`unzip`**, install it:

```bash
sudo apt-get update
sudo apt-get install -y unzip
unzip -v
```

### 3. Install the Edgible CLI globally

```bash
sudo npm install -g @edgible-team/cli
```

Using **`sudo`** with **`-g`** avoids permission errors on stock Ubuntu; alternatively configure **`npm prefix`** to a directory owned by **`ubuntu`** (see npm docs).

### 4. Verify

```bash
edgible --version
edgible --help
```

### 5. First-time sign-in (optional)

Needs outbound **HTTPS** to Edgible:

```bash
edgible auth login
```

More auth options: **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** (*First-time setup*, *Authentication*).

### 6. Install the Edgible agent on this host (optional)

After **`edgible auth login`**, install the agent **with `sudo`** so the CLI can write **systemd** units and paths under **`/etc`**, **`/opt`**, and similar without **permission denied**:

```bash
sudo edgible agent install
```

Omitting **`sudo`** often fails on stock Ubuntu when the installer touches system locations. If the agent misbehaves after install, inspect logs (exact unit name may match CLI output), for example:

```bash
sudo journalctl -u edgible-agent -n 100 --no-pager
```

**Check that the agent is running** (daemon + config the installer recorded). If you installed with **`sudo edgible agent install`**, use **`sudo`** again so the CLI reads the same **root-owned** config paths:

```bash
sudo edgible agent status
```

Add **`--watch`** to refresh when something changes (**Ctrl+C** to stop).

**Follow agent logs** in real time (**`-f`** is short for **`--follow`**; **Ctrl+C** stops following):

```bash
sudo edgible agent logs -f
```

Use the same **`sudo`** as for **`agent install`** / **`agent status`** when the CLI config lives under **root**. Omit **`sudo`** only if you installed and run the CLI entirely as **`ubuntu`**. Tail a fixed number of lines first with **`sudo edgible agent logs -n 200`**; see **`edgible agent logs --help`** for **`--level`**, **`--module`**, and other filters.

This sandbox doc does not replace full self-hosting runbooks; see the **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** for agent details.

### 7. Register the running nginx as an Edgible application (optional)

With **`edgible-demo-nginx` still up** (port **80** mapped as in **Sample web app** above), after **First-time sign-in** and **Install the Edgible agent** (steps **5** and **6**):

```bash
edgible app create existing
```

**`app`** is a short alias for **`application`**; the long form is **`edgible application create existing`**.

The CLI will prompt for an application name, **serving device** (this EC2 host, once the agent has registered it), and—when workload detection runs—a **running local workload** (choose your nginx container) and the **port** to expose (**80** for this demo). Flags for non-interactive use (for example **`--port 80`**, **`--device-id`**, **`--name`**) are documented in **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** and **`edgible application create existing --help`**.

**Inspect the application** you just registered (requires the same **logged-in** CLI session as **`edgible auth login`**):

```bash
edgible app list
edgible app status
```

**`edgible app status`** is an **alias** for **`edgible app get`** (long form **`edgible application get`**). With multiple applications, pass **`--app-id`** (or **`-i`**) with the ID from **`edgible app list`**, or use **`edgible app status --help`** for **`--json`** and other options.

**Troubleshooting — agent log: `Application handler not found` / `Workload status check failed: Handler not found`**

That message means the **serving agent** is trying to health-check an **`appId`** the **Edgible API** knows about, but the agent process has **no local handler** registered for that application (nothing tied your nginx workload to that app on disk yet). In the current **`@edgible-team/cli`** tree, **`edgible app create existing`** creates the application via the **API** but **does not** run the same **“configure local agent for this application”** step that other onboarding flows use—so **`sudo systemctl restart edgible-agent`** alone often **does not** fix reachability.

Until the CLI (or agent sync) wires that up end-to-end: use **`sudo edgible agent logs -f`** and **`edgible device application-health`** (see **`edgible device application-health --help`**) for visibility, confirm **`edgible app status`** shows the app on the right **device**, and follow **internal / support** guidance for your org—or patch the CLI to call local agent configuration after **`create existing`**. Edgible may ship an updated CLI before this note is removed.

## When you are done

1. **Optional:** stop/remove the demo container if it is still running: **`docker stop edgible-demo-nginx`** and **`docker rm edgible-demo-nginx`** (see **Sample web app** above).
2. **Terminate** the EC2 instance (or stop if you will reuse it).
3. **Release** any **Elastic IP**.
4. Review **EBS** volumes and snapshots.
5. Remove unused **security group** rules.

That gives you a **Linux (Ubuntu)**, SSH-first way to validate Docker on EC2 and continue evaluating Edgible from a disposable cloud VM.
