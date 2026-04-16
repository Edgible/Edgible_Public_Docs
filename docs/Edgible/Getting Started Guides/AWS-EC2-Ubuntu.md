# Linux example: Evaluate Edgible self-hosting on disposable AWS EC2 (Ubuntu)

This guide is **Ubuntu LTS on EC2 only**. It mirrors **[AWS-EC2-Windows.md](AWS-EC2-Windows.md)** for people who prefer a **Linux** sandbox: **SSH**, **Docker Engine** with a **small Linux container** you can hit in a browser, then **Node.js**, the **Edgible CLI** (`@edgible-team/cli`), the **Dashboard**, and an optional **application** registration. **Terminate** the instance when finished.

For **Windows Server**, **RDP**, and **Windows containers (IIS)**, use **[AWS-EC2-Windows.md](AWS-EC2-Windows.md)**.

For the **same six steps** **on your Mac** without EC2, see **[Local-MacOS.md](Local-MacOS.md)**.

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

- **Isolation:** Workloads run in AWS, not on your laptop.
- **Low cost / low risk:** **Ubuntu** + **Docker** + **`nginx:alpine`** fit comfortably on a **free-tier–eligible** instance type where your account still has that allowance; always confirm in **[AWS Free Tier](https://aws.amazon.com/free/)** and **Billing**.
- **Clean exit:** **Terminate** the instance; release **Elastic IPs**; remove unused **security groups** and volumes.

## Prerequisites (AWS EC2 — Ubuntu)

### Suggested AMI and instance size

| Goal | Suggested starting point | Notes |
|------|---------------------------|--------|
| **nginx in Docker (quick browser check)** | **Ubuntu Server 22.04 or 24.04 LTS** (Canonical AMI) | **`t3.micro`** is often enough for pull + run; see **root volume** below. |
| **Edgible CLI + agent / heavier self-hosting** | **`t3.small`** or larger, more disk | Follow main Edgible self-hosting docs for production-like sizing. |

**Root volume (EBS) for this guide:** For **Ubuntu + Docker + `nginx:alpine` only**, **20 GiB** is **enough** and a comfortable default (OS, Docker, one tiny image, logs, **`apt`/`npm`** headroom). **8–10 GiB** can work for the bare minimum demo but gets **tight** if you pull **larger** images, keep **Docker** cache, or run **`npm install -g`** with many dependencies—use **≥ 20 GiB** if you are unsure.

**AMI:** In the EC2 launch wizard, search **Ubuntu** and pick **64-bit (x86)** unless you intentionally use **Graviton (arm64)**—then use arm64 AMIs and matching Docker **`arch=`** in the apt repo line (see Docker’s Ubuntu install page).

**Default user:** Canonical AMIs use **`ubuntu`** (not `ec2-user`). Example: **`ssh -i your-key.pem ubuntu@<public-ip>`**.

### Key pair (`.pem`) for SSH login

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

### Connect to the instance

1. Launch (or relaunch) the instance with that **key pair** selected; security group allows **TCP 22** from **your IP**.
2. From your Mac or PC:

   ```bash
   ssh -i /path/to/your-key.pem ubuntu@<instance-public-ip>
   ```

Use **`sudo`** where indicated below (Ubuntu’s **`ubuntu`** user has passwordless sudo).

### Security group (inbound)

- **SSH:** TCP **22** from **your IP** only.
- **HTTP:** TCP **80** from **your IP** (for the nginx demo in a browser).
- **HTTPS:** TCP **443** only if you add TLS or the Edgible docs require it (needed for **`edgible auth login`** outbound is usually allowed by default egress).

---

## 1. Docker (and nginx sample)

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

```bash
docker version
docker run --rm hello-world
```

If **`permission denied`** on **`/var/run/docker.sock`**, you skipped **`usermod`** or your shell does not have the **`docker`** group yet—run **`sudo usermod -aG docker ubuntu`**, then **log out and SSH in again** or **`newgrp docker`**. Until then, **`sudo docker …`** works as a temporary workaround.

### Optional host firewall (UFW)

Only if you enabled **UFW**; many sandboxes rely on the **security group** alone:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw enable   # skip if you do not use UFW
```

### Run nginx (`nginx:alpine`)

```bash
docker run -d --name edgible-demo-nginx -p 80:80 nginx:alpine
```

When **`docker ps`** shows the container **Up**, open **`http://<instance-public-ip>/`** in a browser (security group must allow **TCP 80** from your IP). You should see **“Welcome to nginx!”**

**Logs (optional):** **`docker logs edgible-demo-nginx`**.

**Important:** **Keep `edgible-demo-nginx` running** through steps **4** and **5**. Do not **`docker stop`** / **`docker rm`** until **step 6** (or you intentionally recreate it).

### Verify / test (step 1)

- [ ] **`docker version`** and **`docker run --rm hello-world`** succeed (or **`sudo docker …`** if the group is not active yet).
- [ ] Browser loads **Welcome to nginx!** at **`http://<instance-public-ip>/`**.

---

## 2. Node.js and Edgible CLI

The published package is **`@edgible-team/cli`** (command **`edgible`**). You need **Node.js 16+** and **npm**.

### Node.js LTS (recommended: NodeSource)

Ubuntu’s default **`apt install nodejs`** can be **too old** for the CLI. Use **[NodeSource](https://github.com/nodesource/distributions)** (check their README for the current setup script name), for example:

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs
```

```bash
node --version
npm --version
```

### `unzip` (only if `edgible agent` install complains)

Most Ubuntu images already have **`unzip`**. If the agent install fails on **`unzip`**, install it:

```bash
sudo apt-get update
sudo apt-get install -y unzip
unzip -v
```

### Install the Edgible CLI globally

```bash
sudo npm install -g @edgible-team/cli
```

Using **`sudo`** with **`-g`** avoids permission errors on stock Ubuntu; alternatively configure **`npm prefix`** to a directory owned by **`ubuntu`** (see npm docs).

### Verify / test (step 2)

```bash
edgible --version
edgible --help
```

- [ ] **`node --version`** shows **16+**.
- [ ] **`edgible --version`** prints a version string.

---

## 3. Edgible sign-up and `edgible auth login`

### 3.1 Browser: create or open your account (Dashboard)

On your **local laptop or desktop** (the machine with a browser—not only over SSH if you prefer a GUI):

1. Open **[https://www.edgible.com](https://www.edgible.com)** (or **[https://www.edgible.com/signup](https://www.edgible.com/signup)**).
2. Click **Login** (typically **top right** on the public site).
3. Choose **Sign up here** (or equivalent) for a **new** account, or **sign in** if you already have one.
4. After success you should land on the **Dashboard**. Keep it open for **steps 4** and **5** to compare with the CLI on the EC2 host.

**Verify / test (3.1)**

- [ ] You see the **Dashboard** while logged in.

### 3.2 CLI on the Ubuntu host: authenticate

Needs outbound **HTTPS** from the instance (default security group egress usually allows this):

```bash
edgible auth login
```

**Verify / test (3.2)**

- [ ] **`edgible auth login`** completes without errors.

More auth options: **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** (*First-time setup*, *Authentication*).

---

## 4. Start the Edgible agent

After **`edgible auth login`**, install the agent **with `sudo`** so the CLI can write **systemd** units and paths under **`/etc`**, **`/opt`**, and similar:

```bash
sudo edgible agent install
```

**Dashboard:** Refresh the **Dashboard** and look for this **EC2 instance** as a **device** (hostname / labels depend on Edgible UI).

If the agent misbehaves after install:

```bash
sudo journalctl -u edgible-agent -n 100 --no-pager
sudo edgible agent status
```

Add **`--watch`** on **`agent status`** (**Ctrl+C** to stop). Follow logs: **`sudo edgible agent logs -f`** (**Ctrl+C** to stop). Use **`sudo`** consistently if install wrote root-owned config. See **`edgible agent logs --help`**.

This sandbox doc does not replace full self-hosting runbooks; see the **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** for agent details.

### Verify / test (step 4)

- [ ] **`sudo edgible agent status`** shows a healthy agent.
- [ ] **Dashboard** reflects this **device** in line with the CLI.

---

## 5. Create an Edgible application (from the running container)

With **`edgible-demo-nginx` still up** (port **80** mapped as in **step 1**), after **steps 3–4**:

```bash
edgible app create existing
```

**`app`** is a short alias for **`application`**; the long form is **`edgible application create existing`**.

The CLI prompts for an **application name**, **serving device** (this EC2 host, once the agent has registered it), **running workload** (choose **`edgible-demo-nginx`**), and **port** (**80** for this demo). Flags (for example **`--port 80`**, **`--device-id`**, **`--name`**) are in the **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)** and **`edgible application create existing --help`**.

**Dashboard:** Confirm the new **application** appears and matches **`edgible app list`** / **`edgible app status`**.

```bash
edgible app list
edgible app status
```

**`edgible app status`** is an **alias** for **`edgible app get`**. With multiple applications, pass **`--app-id`** (or **`-i`**) from **`edgible app list`**.

### Troubleshooting — agent log: `Application handler not found` / `Workload status check failed: Handler not found`

That message means the **serving agent** is trying to health-check an **`appId`** the **Edgible API** knows about, but the agent process has **no local handler** registered for that application yet. In the current **`@edgible-team/cli`** tree, **`edgible app create existing`** creates the application via the **API** but **does not** always run the same **“configure local agent for this application”** step that other onboarding flows use—so **`sudo systemctl restart edgible-agent`** alone often **does not** fix reachability.

Until the CLI (or agent sync) wires that up end-to-end: use **`sudo edgible agent logs -f`** and **`edgible device application-health`** (see **`edgible device application-health --help`**) for visibility, confirm **`edgible app status`** shows the app on the right **device**, and follow **internal / support** guidance for your org—or patch the CLI to call local agent configuration after **`create existing`**. Edgible may ship an updated CLI before this note is removed.

### Verify / test (step 5)

- [ ] **`edgible app ls`** (same command as **`edgible app list`**) lists your demo app. Note the **Serving IP**, **Port**, **Protocol**, and the **URL** line in the output (or run **`edgible app ls --json`** if you need machine-readable fields).
- [ ] **Dashboard** shows the **application** in a way that matches the CLI.
- [ ] **Internet (off-LAN):** On a **device that is not on the same LAN as your laptop** (for example a **smartphone on cellular/mobile data** with **Wi‑Fi off** or **not** on your home/office Wi‑Fi), open a browser and go to the **URL** from **`edgible app ls`**, or **`http://` / `https://`** + **Serving IP** + **`:`** + **Port** as printed. You should see the same **nginx** welcome page as when you opened **`http://<instance-public-ip>/`** in step **1**, proving Edgible is **hosting the workload on the internet**, not only inside your VPC or local browser session. If it fails at first, wait for routing to propagate, check **`edgible app status`**, and retry.

---

## 6. Teardown (Edgible-related; EC2 cleanup)

### 6.1 Application (if created in step 5)

```bash
edgible app delete
```

The CLI shows a **pick list** of applications; choose the one you created in step **5**. See **`edgible app delete --help`** for non-interactive flags (for example **`--app-id`**) if you need scripting.

### 6.2 Agent

```bash
sudo edgible agent uninstall --remove-files
```

Confirm flags in **[Edgible CLI user guide](../../../../Edgible_Docs/Website/EDGIBLE_CLI_USER_GUIDE.md)**.

### 6.3 CLI session and package

```bash
edgible auth logout
sudo npm uninstall -g @edgible-team/cli
```

### 6.4 Demo container

```bash
docker stop edgible-demo-nginx
docker rm edgible-demo-nginx
```

### 6.5 Optional: remove Docker Engine

Only if you want the instance clean of Docker (not required before terminate):

```bash
sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Follow Docker’s docs for leftover **`/var/lib/docker`** if you need disk reclaimed.

### 6.6 EC2 and AWS cleanup

1. **Terminate** the EC2 instance (or stop if you will reuse it).
2. **Release** any **Elastic IP**.
3. Review **EBS** volumes and snapshots.
4. Remove unused **security group** rules.

**Dashboard:** After teardown, refresh the **Dashboard**; device/application rows should update as Edgible’s backend syncs.

### Verify / test (step 6)

- [ ] Instance **terminated** (or stopped) and billing risk understood.
- [ ] No demo container / agent / CLI left **if** you intended full cleanup.

That gives you a **Linux (Ubuntu)**, SSH-first path through the same **six steps** as **[Local-MacOS.md](Local-MacOS.md)** and **[AWS-EC2-Windows.md](AWS-EC2-Windows.md)**.
