# Production Deployment

Deploy Frappe **V15** as a bare-metal install on Ubuntu 22.04, with MariaDB on the
same host and Cloudflare proxying in front. Each subdomain (`a.example.com`,
`b.example.com`, …) is its own Frappe site under a single bench — DNS-based
multitenancy. This guide assumes a fresh Ubuntu 22.04 server you can reach over SSH
as `root`, a domain managed in Cloudflare, and familiarity with the Linux command
line. See the [conventions and placeholders](README.md#conventions).

```
             ┌──────────────────────────────┐
             │   Cloudflare (proxied DNS)   │
             │   Origin Cert + Full Strict  │
             └──────────────┬───────────────┘
                            │ HTTPS (443)
                            ▼
┌───────────────────────────────────────────────┐
│          Cloud server (Ubuntu 22.04)          │
│   Nginx — reverse proxy, SSL termination      │
│   ├── a.example.com ──► gunicorn :8000        │
│   ├── b.example.com ──► gunicorn :8000        │
│   └── c.example.com ──► gunicorn :8000        │
│   Supervisor — gunicorn, workers,             │
│                scheduler, socketio            │
│   MariaDB + Redis (localhost only)            │
└───────────────────────────────────────────────┘
```

Nginx matches each request's `Host` header against `sites/<site-name>/`. Every site
shares one bench, one MariaDB, and one Redis.

Commands run as the `frappe` user unless they carry a `sudo` prefix or a step says
otherwise. The bench root is `~/frappe/frappe-bench` (absolute:
`/home/frappe/frappe/frappe-bench`). The app tags shown below are samples — confirm
the current ones on each app's releases page before you run them.

| Component   | Version |
| ----------- | ------- |
| OS          | Ubuntu 22.04 LTS |
| Python      | 3.10 (the 22.04 system default) |
| Node.js     | 18 (via nvm) |
| MariaDB     | from `apt` (10.6 on 22.04) |
| Redis       | from `apt` |
| wkhtmltopdf | 0.12.6.1-3 patched-Qt build |

## Contents

1. [Provision the server](#1-provision-the-server)
2. [Prepare the operating system](#2-prepare-the-operating-system)
3. [Install MariaDB](#3-install-mariadb)
4. [Install Redis and wkhtmltopdf](#4-install-redis-and-wkhtmltopdf)
5. [Install the toolchain](#5-install-the-toolchain)
6. [Initialize the bench and apps](#6-initialize-the-bench-and-apps)
7. [Enable production mode](#7-enable-production-mode)
8. [Create and configure sites](#8-create-and-configure-sites)
9. [Set up Cloudflare SSL](#9-set-up-cloudflare-ssl)
10. [Configure the edge and Nginx](#10-configure-the-edge-and-nginx)
11. [Point DNS and verify](#11-point-dns-and-verify)
12. [Back up](#12-back-up)
13. [Update apps](#13-update-apps)
14. [Routine operations](#14-routine-operations)
15. [Harden the server](#15-harden-the-server)
16. [Troubleshooting](#16-troubleshooting)

## 1. Provision the server

Size the instance, then lock the network down to only the ports the deployment
needs.

Provision an Ubuntu 22.04 LTS instance. 2 vCPU / 4 GB works but is tight — turn on
swap ([step 2](#2-prepare-the-operating-system)) and avoid heavy background jobs.
Put the database on a separate SSD-tier data disk; Frappe is I/O-sensitive.

| Resource | Minimum | Comfortable |
| -------- | ------- | ----------- |
| vCPU | 2 | 4 |
| RAM | 4 GB | 8 GB |
| System disk | 40 GB ESSD | 60 GB ESSD |
| Data disk (DB) | 40 GB ESSD | 100 GB ESSD |
| Image | Ubuntu 22.04 LTS 64-bit | Same |

Configure the inbound security group to allow only what's needed. Pinning 80 and 443
to Cloudflare's ranges ([IPv4](https://www.cloudflare.com/ips-v4),
[IPv6](https://www.cloudflare.com/ips-v6)) stops attackers from bypassing Cloudflare
and hitting the origin directly.

| Port | Source | Purpose |
| ---- | ------ | ------- |
| 22 | Your IP only | SSH |
| 80 | Cloudflare IP ranges | HTTP → HTTPS redirect |
| 443 | Cloudflare IP ranges | HTTPS |
| All others | Deny | — |

<a id="refresh-allowlist"></a>Refresh the Cloudflare allowlist quarterly — Cloudflare
adds ranges occasionally.

> ⚠️ Never expose these ports publicly: 3306 (MariaDB), 6379 (Redis), 8000
> (gunicorn), 9000 (SocketIO), 11000–13000 (bench Redis). They must be reachable
> only from `localhost`.

## 2. Prepare the operating system

On first login to the fresh instance, **everything in this section runs as `root`**
(no `sudo` prefix needed), until you switch to the `frappe` user at the end.

Refresh the package index, upgrade installed packages, and install the operational
tools. `fail2ban` auto-bans SSH brute-force attempts:

```bash
apt update
apt upgrade -y
apt install -y fail2ban htop tmux vim curl wget git unzip
```

Install the build dependencies — the C toolchain, the headers Python C-extensions
compile against, and the X and font libraries wkhtmltopdf needs:

```bash
apt install -y \
  git curl wget build-essential \
  pkg-config libssl-dev libffi-dev \
  zlib1g-dev software-properties-common \
  xvfb libfontconfig xfonts-75dpi
```

Set the timezone to your own — it drives log timestamps and cron schedules. List the
options with `timedatectl list-timezones`:

```bash
timedatectl set-timezone Asia/Singapore
```

Add swap on 4 GB instances; Frappe leans hard on memory during builds and
migrations. Skip this on 8 GB or larger unless you expect heavy pressure. The
`fstab` line restores swap after reboot, and `swappiness=10` keeps the kernel on RAM
(the default of 60 swaps too eagerly for a database host):

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
sysctl vm.swappiness=10
echo 'vm.swappiness=10' >> /etc/sysctl.conf
```

Create the `frappe` user to run the bench under, grant it passwordless sudo for now,
and switch to it:

```bash
adduser frappe
usermod -aG sudo frappe
echo "frappe ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/frappe
chmod 755 /home/frappe
su - frappe
```

- `NOPASSWD:ALL` is convenient during setup; tighten it later in
  [Harden the server](#15-harden-the-server).
- `chmod 755 /home/frappe` is **critical** — Nginx (`www-data`) must traverse this
  directory to serve static assets, or every asset returns 403.

From here on, everything runs as `frappe` unless it carries `sudo`.

## 3. Install MariaDB

Frappe stores its data in MariaDB. Install it, harden it, give root a password, and
apply Frappe's required configuration.

Install the server and client packages, then run the hardening script. Choose a
strong root password and answer **Y** to every prompt:

```bash
sudo apt install -y mariadb-server mariadb-client libmariadb-dev
sudo mariadb-secure-installation
```

Ubuntu authenticates MariaDB root through `unix_socket`, so that password only works
under `sudo mariadb`. `bench new-site` connects with a password instead, so set one
— keep it, because you need it for every `bench new-site` and for backups:

```bash
sudo mariadb \
  -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'YOUR_ROOT_PW';" \
  -e "FLUSH PRIVILEGES;"
```

Apply Frappe's required tuning as a drop-in config file, then restart MariaDB to
load it:

```bash
sudo tee /etc/mysql/mariadb.conf.d/99-frappe.cnf > /dev/null <<'EOF'
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

innodb-buffer-pool-size = 2G
max_allowed_packet = 256M

bind-address = 127.0.0.1

[mysql]
default-character-set = utf8mb4

[mysqldump]
default-character-set = utf8mb4
EOF

sudo systemctl restart mariadb
```

- The `utf8mb4` block is required. Without it, `bench new-site` fails with
  `Specified key was too long`.
- `innodb-buffer-pool-size = 2G` is InnoDB's RAM cache; trim it on smaller
  instances.
- `bind-address = 127.0.0.1` confines MariaDB to `localhost`; with the security
  group, that keeps it off the internet.

## 4. Install Redis and wkhtmltopdf

Redis backs Frappe's cache, queues, and real-time messaging. Install it and enable
it on boot:

```bash
sudo apt install -y redis-server
sudo systemctl enable redis-server
```

> 💡 This system Redis (port 6379) is separate from the three Redis instances bench
> starts later on ports 11000 (cache), 12000 (queue), and 13000 (socketio) via
> `bench setup production`, all bound to `localhost`.

Frappe renders PDFs with the **patched-Qt** build of wkhtmltopdf; the distribution
package does not work. Install the prebuilt `.deb` and check its version:

```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_amd64.deb
sudo dpkg -i wkhtmltox_0.12.6.1-3.jammy_amd64.deb
sudo apt install -f -y
wkhtmltopdf --version
```

wkhtmltopdf is the correct build when the version output contains `with patched qt`.
If it does not, PDF generation fails at runtime — see
[Troubleshooting](#16-troubleshooting).

## 5. Install the toolchain

Production uses the **system Python with `pip`** (not `uv`), because some bench
operations run under `sudo`, and both `frappe` and `root` must resolve the same
`bench` binary at `/usr/local/bin/bench`.

Install the Python toolchain and Bench. `bench --version` and `sudo bench --version`
must print the same version, confirming both users resolve the same binary:

```bash
sudo apt install -y python3-dev python3-pip python3-venv
sudo pip3 install frappe-bench
bench --version
sudo bench --version
```

Install Node 18 via nvm, plus Yarn. `nvm alias default 18` makes Node 18 the default
for new shells:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.5/install.sh | bash
source ~/.bashrc
nvm install 18
nvm alias default 18
npm install -g yarn
```

## 6. Initialize the bench and apps

A *bench* is a self-contained Frappe project: its own Python virtualenv, its own
`node_modules`, and one or more sites under it.

Create the parent directory and initialize the bench on the V15 branch. This clones
Frappe into `apps/frappe`, builds the virtualenv at `env/`, installs dependencies,
and runs an initial build (a few minutes):

```bash
mkdir -p ~/frappe
cd ~/frappe
bench init frappe-bench --frappe-branch version-15 --verbose
```

Fetch the apps the bench will host. `bench get-app` clones and registers each app
but installs it on **no site yet**:

```bash
cd ~/frappe/frappe-bench
bench get-app --branch version-15 erpnext
bench get-app --branch version-15 hrms
```

A custom public app follows the same pattern:

```bash
bench get-app --branch main https://github.com/<org>/<custom-app>.git
```

Pin `setuptools<81` (mandatory). A transitive dependency (`googleapiclient`) imports
`pkg_resources`, removed in setuptools 81+, so without this `bench new-site` fails
with `ModuleNotFoundError: No module named 'pkg_resources'`. The second command
prints `OK` when the pin took:

```bash
./env/bin/pip install "setuptools<81"
./env/bin/python -c "import pkg_resources; print('OK')"
```

> 💡 `env/` is per-bench — apply this pin in every bench on the host.

### Optional: pin apps to release tags

By default each app tracks a moving branch (`version-15`), so `bench update` pulls
whatever sits at its head. Pin to tags when you want deliberate version bumps or
matching versions across environments. Find the current stable tags on each app's
releases page — the tags below are samples on the V15 line; always confirm the
current ones before you pin.

| App     | Sample stable tag (V15 line) | Releases |
| ------- | ---------------------------- | -------- |
| Frappe  | `v15.58.0` | [github.com/frappe/frappe/releases](https://github.com/frappe/frappe/releases) |
| ERPNext | `v15.54.3` | [github.com/frappe/erpnext/releases](https://github.com/frappe/erpnext/releases) |
| HRMS    | `v15.41.0` | [github.com/frappe/hrms/releases](https://github.com/frappe/hrms/releases) |

Check out the chosen tag in each app, then refresh requirements. `bench setup
requirements` picks up anything differing between the branch head and the tag:

```bash
cd apps/frappe
git fetch --depth 1 upstream tag v15.58.0
git checkout v15.58.0
cd ../erpnext
git fetch --depth 1 upstream tag v15.54.3
git checkout v15.54.3
cd ../hrms
git fetch --depth 1 upstream tag v15.41.0
git checkout v15.41.0

cd ~/frappe/frappe-bench
bench setup requirements
```

Each app is pinned when `git describe --tags` reports the tag you checked out. If
checkout fails on `yarn.lock`, see [Troubleshooting](#16-troubleshooting).

### Optional: add a private app

For private GitHub repos, use a read-only **deploy key** — it is per-repo and
carries no personal credentials. As `frappe`, generate the key, pressing Enter twice
when prompted for no passphrase, then configure SSH to use it for `github.com`:

```bash
ssh-keygen -t ed25519 -C "frappe@ecs-production" -f ~/.ssh/github_deploy

cat >> ~/.ssh/config <<'EOF'
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_deploy
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config ~/.ssh/github_deploy
chmod 644 ~/.ssh/github_deploy.pub
```

Show the public key with `cat ~/.ssh/github_deploy.pub` and add it in GitHub under
**Repo → Settings → Deploy keys → Add deploy key**, with **Allow write access off**
— `bench get-app` only reads. Test the key, then clone the app:

```bash
ssh -T git@github.com
cd ~/frappe/frappe-bench
bench get-app --branch main git@github.com:<github-org>/your-private-app.git
```

The key works when `ssh -T git@github.com` answers `Hi <name>! You've successfully
authenticated...`.

> 💡 A deploy key works for one repo. With several private apps, issue a key per repo
> (each with its own `Host` alias) or use a GitHub machine user.

## 7. Enable production mode

Move off `bench start` onto the Supervisor and Nginx process model.

> ⚠️ Order matters. Some apps' install patches reach for the bench Redis instances.
> If you create sites before this step, `bench new-site` throws
> `ConnectionRefusedError` on 11000–13000. Run `sudo bench setup production frappe
> --yes` first, then create sites in [step 8](#8-create-and-configure-sites).

Install Supervisor and Nginx, turn on DNS multitenancy, then set up production.
Enabling multitenancy **before** `bench setup production` makes the generated Nginx
config route by `Host` header:

```bash
sudo apt install -y supervisor nginx

cd ~/frappe/frappe-bench
bench config dns_multitenant on
sudo bench setup production frappe --yes
```

`bench setup production` generates the Nginx and Supervisor configs, sets up
fail2ban rules, and starts the bench's Redis instances (11000/12000/13000).
Production mode is up when every process reads `RUNNING` — expect
`frappe-bench-redis-cache/-queue/-socketio`, `-frappe-web`, `-frappe-schedule`, the
three `-worker` processes, and `-node-socketio`:

```bash
sudo supervisorctl status
```

## 8. Create and configure sites

Create one Frappe site per subdomain, each under `sites/<site-name>/`.

> ⚠️ Name each site **exactly** as its subdomain. Frappe routes by matching the
> `Host` header against `sites/<site-name>/`, so `a.example.com` →
> `sites/a.example.com/`.

Create each site, installing its apps in dependency order. `--install-app` installs
in the order listed, so put each dependency ahead of what needs it (HRMS needs
ERPNext). `--db-name` is an optional readable name; omit it for a hash. Save each
`--admin-password`:

```bash
cd ~/frappe/frappe-bench

bench new-site a.example.com \
  --mariadb-root-password 'YOUR_ROOT_PW' \
  --admin-password 'YOUR_ADMIN_PW' \
  --db-name a_example \
  --install-app erpnext \
  --install-app hrms

bench new-site b.example.com \
  --mariadb-root-password 'YOUR_ROOT_PW' \
  --admin-password 'YOUR_ADMIN_PW' \
  --db-name b_example \
  --install-app erpnext
```

A new site starts with background jobs **off**. Enable the scheduler per site:

```bash
bench --site a.example.com enable-scheduler
bench --site b.example.com enable-scheduler
```

> 💡 Server scripts ship off in V15. Turn them on only if your apps need them:
> `bench --site a.example.com set-config server_script_enabled 1` (drop `--site` and
> add `-g` to cover all sites).

## 9. Set up Cloudflare SSL

Production uses **Full (Strict)** — HTTPS to the origin, with Cloudflare validating
the origin certificate.

> ⚠️ Never use Flexible for ERPNext or anything handling authentication or business
> data. Login cookies, form posts, and API tokens would cross from Cloudflare to
> your origin in the clear.

Generate an Origin Certificate in the Cloudflare Dashboard → **your domain →
SSL/TLS → Origin Server → Create Certificate**, with hostnames `*.example.com` and
`example.com` (the wildcard covers future subdomains) and 15-year validity.

> ⚠️ Save both the certificate and the private key immediately — Cloudflare reveals
> the private key only once.

Install the certificate on the server. Paste the Origin Certificate into
`origin.pem` and the Private Key into `origin.key` when each `nano` opens, then fix
the permissions:

```bash
sudo mkdir -p /etc/ssl/cloudflare
sudo chmod 700 /etc/ssl/cloudflare

sudo nano /etc/ssl/cloudflare/origin.pem
sudo nano /etc/ssl/cloudflare/origin.key

sudo chmod 600 /etc/ssl/cloudflare/origin.key
sudo chmod 644 /etc/ssl/cloudflare/origin.pem
```

Merge a `wildcard` block into `sites/common_site_config.json` (keep existing keys)
so `bench setup nginx` generates an SSL server block for every subdomain:

```json
{
  "wildcard": {
    "domain": "*.example.com",
    "ssl_certificate": "/etc/ssl/cloudflare/origin.pem",
    "ssl_certificate_key": "/etc/ssl/cloudflare/origin.key"
  }
}
```

Then set the dashboard mode under **SSL/TLS → Overview → Full (strict)**. Continue
to [Configure the edge and Nginx](#10-configure-the-edge-and-nginx) to apply it.

<details>
<summary><strong>Alternatives (non-production)</strong></summary>

**Full (self-signed)** — internal or non-sensitive use. Same as above, but generate
the certificate yourself instead of from Cloudflare, then set the mode to **Full**:

```bash
sudo mkdir -p /etc/ssl/cloudflare
sudo chmod 700 /etc/ssl/cloudflare
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /etc/ssl/cloudflare/origin.key \
  -out /etc/ssl/cloudflare/origin.pem \
  -subj "/CN=*.example.com"
sudo chmod 600 /etc/ssl/cloudflare/origin.key
sudo chmod 644 /etc/ssl/cloudflare/origin.pem
```

**Flexible (throwaway dev only)** — Cloudflare reaches the origin over plain HTTP on
port 80. Swap the 443 security-group rule for 80 (still locked to Cloudflare IPs),
add **no** `wildcard` block, set `host_name` per site so Frappe still emits
`https://` URLs, and set the dashboard mode to **Flexible**:

```bash
bench --site a.example.com set-config host_name "https://a.example.com"
```

</details>

## 10. Configure the edge and Nginx

This applies to every SSL mode.

In the Cloudflare Dashboard → **SSL/TLS → Edge Certificates**, turn on **Always Use
HTTPS** and **Automatic HTTPS Rewrites**, and set **Minimum TLS Version** to 1.2.

Bench's generated Nginx config references a log format named `main` that Ubuntu's
default `nginx.conf` never defines — without it, Nginx won't start. Add this inside
the `http { ... }` block of `/etc/nginx/nginx.conf`:

```
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
```

Regenerate the Nginx config from the current sites and the `wildcard` block, test
it, and reload:

```bash
cd ~/frappe/frappe-bench
bench setup nginx --yes
sudo nginx -t
sudo systemctl reload nginx
```

The config is valid when `sudo nginx -t` reports `syntax is ok` and `test is
successful`. If it fails, fix the error before you reload.

## 11. Point DNS and verify

In the Cloudflare Dashboard → **DNS → Records → Add record**, add one A record per
subdomain. `Name` is the subdomain part only (`a` → `a.example.com`):

| Type | Name | Content | Proxy |
| ---- | ---- | ------- | ----- |
| A | a | `<server public IP>` | Proxied (orange) |
| A | b | `<server public IP>` | Proxied (orange) |

> 💡 The proxy toggle must be orange. Grey (DNS-only) routes straight to the origin —
> no SSL termination, no WAF, and your origin IP exposed.

Verify the services and each subdomain. Every `frappe-bench-*` process should read
`RUNNING`, Nginx and MariaDB `active (running)`, and each `curl` should return
`HTTP/2 200`:

```bash
sudo supervisorctl status
sudo systemctl status nginx
sudo systemctl status mariadb

curl -I https://a.example.com
curl -I https://b.example.com
```

Then open each subdomain and sign in as `Administrator`. For 5xx or TLS errors, see
[Troubleshooting](#16-troubleshooting).

## 12. Back up

Run three layers of backup, each covering a different failure: local backups handle
site corruption; offsite copies handle losing the host or account; disk snapshots
are the quickest recovery from OS-level damage.

### Manual backup

Run one ahead of anything risky — updates, OS upgrades, schema changes. The first
command backs up one site; `bench backup-all-sites` covers every site. `--with-files`
adds tarballs of public and private files alongside the database dump, which you
almost always want. Backups land in `sites/<site>/private/backups/`:

```bash
cd ~/frappe/frappe-bench
bench --site a.example.com backup --with-files
bench backup-all-sites --with-files
```

### Scheduled backup

The schedule runs under `cron` (`sudo apt install -y cron` if it is missing). Edit
the `frappe` user's crontab with `crontab -e` and add a daily backup at 02:00 and a
Sunday 04:00 prune of backups older than 14 days (tune `-mtime +14` to your
retention):

```
0 2 * * * cd /home/frappe/frappe/frappe-bench && /usr/local/bin/bench backup-all-sites --with-files >> /home/frappe/backup.log 2>&1
0 4 * * 0 find /home/frappe/frappe/frappe-bench/sites/*/private/backups -type f -mtime +14 -delete
```

The schedule is active when `crontab -l` lists both lines and
`tail -20 ~/frappe/backup.log` shows a recent run.

### Offsite to Alibaba OSS

Install ossutil v1 and configure it. `ossutil64 config` prompts for the AccessKey
ID, secret, and endpoint:

```bash
curl -O https://gosspublic.alicdn.com/ossutil/1.7.18/ossutil64
chmod 755 ossutil64
sudo mv ossutil64 /usr/local/bin/
ossutil64 config
```

> 💡 These commands use ossutil v1 (`ossutil64`).
> [ossutil 2.x](https://www.alibabacloud.com/help/en/oss/developer-reference/ossutil-overview/)
> changes the binary name, download URL, and config flow; the v1 commands here still
> work.

> ⚠️ Use a dedicated RAM user (for example `frappe-backup-writer`) whose policy
> grants only **`oss:PutObject`**, **`oss:GetObjectMeta`**, and **`oss:ListObjects`**
> on the backup bucket — no `GetObject`, no `DeleteObject`, no wildcards. A
> compromised server can then push new objects but can't read or delete historical
> backups. (`GetObjectMeta` and `ListObjects` let `-u` incremental sync compare
> timestamps.) Feed this user's AccessKey into `ossutil64 config`.

Schedule the upload to run after the 02:00 local backup. cron's `PATH` omits
`/usr/local/bin`, hence the absolute path; `-u` uploads only new or changed files:

```
30 3 * * * for d in /home/frappe/frappe/frappe-bench/sites/*/private/backups; do site=$(basename $(dirname $(dirname $d))); /usr/local/bin/ossutil64 cp -r -u "$d" "oss://your-backup-bucket/$site/"; done >> /home/frappe/oss-backup.log 2>&1
```

The upload is working when `/home/frappe/oss-backup.log` shows `Succeed: total ...`.

### Disk snapshots

In the ECS Console → **Storage & Snapshots → Snapshot Policies → Create**, attach a
policy to the system and data disks: daily at 04:00, 7-day retention. Snapshots
restore a whole disk in minutes (the fastest path from a boot failure or accidental
`rm -rf`), but they live in the same account — OSS replication is what covers
account-level disasters.

### Rehearse the restore

<a id="rehearse-restore"></a>A backup you've never restored isn't a backup. Each
quarter, download a backup from OSS (proving the offsite copy is usable), stand up a
staging instance on the same Frappe and app versions, restore, then sign in and
spot-check recent data. Write down the exact commands that worked:

```bash
bench --site <staging-site> restore <path-to-db-dump.sql.gz> \
  --with-public-files <path-to-files.tar> \
  --with-private-files <path-to-private-files.tar>
```

The restore succeeds when you can sign in to the staging site and recent records are
present.

## 13. Update apps

How you update depends on whether your apps track a branch or are pinned to tags.

### Tracking a branch (default)

`bench update` backs up every site, pulls every app, refreshes dependencies, builds
assets, migrates every site, and restarts services:

```bash
cd ~/frappe/frappe-bench
bench update
```

To migrate a single site after a manual change, run
`bench --site a.example.com migrate`.

### Pinned to tags

Move each app onto its target tag, then rebuild and migrate (the target tags below
are samples):

```bash
cd ~/frappe/frappe-bench
bench backup-all-sites --with-files

cd ~/frappe/frappe-bench/apps/frappe
git fetch --depth 1 upstream tag v15.59.0
git checkout v15.59.0
cd ../erpnext
git fetch --depth 1 upstream tag v15.55.0
git checkout v15.55.0
cd ../hrms
git fetch --depth 1 upstream tag v15.43.0
git checkout v15.43.0

cd ~/frappe/frappe-bench
bench setup requirements
bench build
bench --site all migrate
bench restart
```

The update is complete when `bench version` shows each app on its new tag.

## 14. Routine operations

### Add a subdomain

Add a Proxied A record (`c` → server IP), then create and wire up the site. The
wildcard certificate already covers the new subdomain:

```bash
cd ~/frappe/frappe-bench
bench new-site c.example.com \
  --mariadb-root-password 'YOUR_ROOT_PW' \
  --admin-password 'YOUR_ADMIN_PW' \
  --install-app erpnext

bench setup nginx --yes
sudo systemctl reload nginx
bench --site c.example.com enable-scheduler
curl -I https://c.example.com
```

The subdomain is live when the `curl` returns `HTTP/2 200` before you log in. In
Flexible mode, also set
`bench --site c.example.com set-config host_name "https://c.example.com"`.

### Restart services

`bench restart` restarts everything; the targeted `supervisorctl restart` hits only
the web workers. Use the web-only restart after changing HTTP-side Python code, to
avoid killing long-running background jobs:

```bash
bench restart
sudo supervisorctl restart frappe-bench-web:frappe-bench-frappe-web
```

### Find the logs

`web.error.log` and `worker.error.log` are where most investigations begin
(`sudo tail -f <path>`).

| What | Path |
| ---- | ---- |
| Bench (web, worker, scheduler, socketio) | `~/frappe/frappe-bench/logs/` |
| Nginx | `/var/log/nginx/access.log`, `/var/log/nginx/error.log` |
| Supervisor | `/var/log/supervisor/supervisord.log` |
| MariaDB | `/var/log/mysql/error.log` |

### Renew the Origin Certificate

The Origin Certificate is valid 15 years, so this is rare. Regenerate it in
Cloudflare with the same hostnames, overwrite `/etc/ssl/cloudflare/origin.{pem,key}`,
fix permissions (`644` pem, `600` key), then `sudo systemctl reload nginx`. Verify
the expiry on the server itself — port 443 is firewalled to Cloudflare, and the
proxy would show the edge cert:

```bash
echo | openssl s_client -showcerts -servername a.example.com -connect 127.0.0.1:443 2>/dev/null | openssl x509 -noout -dates
```

## 15. Harden the server

Work through this checklist before calling the deployment production-ready, and
revisit it quarterly. Each item carries a check you can run.

- [ ] **Disable root SSH login** — set `PermitRootLogin no` in
  `/etc/ssh/sshd_config`, then `sudo systemctl reload ssh`. Check:
  `sudo sshd -T | grep permitrootlogin` shows `no`.
- [ ] **SSH keys only** — set `PasswordAuthentication no` in the same file. **Prove
  key login works first**, or you lock yourself out. Check:
  `sudo sshd -T | grep passwordauthentication` shows `no`.
- [ ] **Tighten the `frappe` sudoers** — `sudo bench setup sudoers frappe` replaces
  `NOPASSWD:ALL` with narrower rules. Check: `/etc/sudoers.d/frappe` no longer
  contains `NOPASSWD:ALL`.
- [ ] **fail2ban running** — check: `sudo fail2ban-client status` lists the enabled
  jails.
- [ ] **Cloudflare WAF enabled** — check: a managed ruleset is deployed under
  Security → WAF → Managed rules.
- [ ] **Cloudflare rate limiting** — check: a rule covering `/login` and `/api/*`
  exists under Security → WAF → Rate limiting rules.
- [ ] **Developer mode off** — `bench --site <site> set-config developer_mode 0` per
  site. Check: `bench --site <site> get-config developer_mode` returns `0`.
- [ ] **Copy each site's `encryption_key`** — see below.
- [ ] **Audit installed apps** — check: `bench version` per site; remove anything
  unused.
- [ ] **Patch the OS monthly** — `sudo apt update`, then `sudo apt upgrade -y`, or
  enable `unattended-upgrades`. Check: `/var/log/apt/history.log` shows an upgrade
  within the month (or `unattended-upgrades --dry-run` succeeds).
- [ ] **Refresh the Cloudflare IP allowlist** — quarterly; see
  [Provision the server](#refresh-allowlist).
- [ ] **Rehearse a restore** — quarterly; see
  [Rehearse the restore](#rehearse-restore).

**The encryption key.** Each site's `sites/<site>/site_config.json` holds a unique
`encryption_key` that encrypts password fields and stored API tokens. Losing it
doesn't stop the site, but **every encrypted field becomes permanently
unrecoverable**. Keep a copy of each site's key in your secret manager, separate from
the database backups.

## 16. Troubleshooting

Start with `~/frappe/frappe-bench/logs/` and `/var/log/nginx/error.log`.

- **`ModuleNotFoundError: No module named 'pkg_resources'`** during `bench new-site`
  — pin setuptools: `./env/bin/pip install "setuptools<81"`.
- **`ConnectionRefusedError` on 127.0.0.1:11000/12000/13000** during `bench new-site`
  — bench Redis isn't running; production mode wasn't enabled. Check
  `sudo supervisorctl status`, then `sudo bench setup production frappe --yes`.
- **`bench new-site` rejects every MariaDB password** — root is still on
  `unix_socket` auth; re-run the `ALTER USER` from [step 3](#3-install-mariadb).
- **`wkhtmltopdf --version` doesn't show "with patched qt"** — the wrong build is
  installed. Run `sudo apt remove wkhtmltopdf`, then redo
  [step 4](#4-install-redis-and-wkhtmltopdf).
- **Tag checkout fails on `yarn.lock`** (`error: Your local changes ... would be
  overwritten`) — run `git checkout .`, then `git checkout <tag>`.
- **"Internal Server Error" after `bench update`** — the dependency refresh didn't
  finish: run `bench setup requirements`, then `bench restart`.
- **Site loads but assets 404** — stale build, or `/home/frappe` lost traversal
  permission: run `bench build`, then `sudo systemctl reload nginx`. Still failing?
  Confirm `ls -ld /home/frappe` shows `drwxr-xr-x`; if not, `sudo chmod 755
  /home/frappe`.
- **502 Bad Gateway after reboot** — a bench service didn't come back. Run
  `sudo supervisorctl status`, then investigate anything `FATAL` or `STOPPED` in the
  logs.
- **Wrong site loads on a subdomain** — DNS multitenancy is off, or Nginx wasn't
  regenerated after adding the site: run `bench config dns_multitenant on`, then
  `bench setup nginx --yes`, then `sudo systemctl reload nginx`.
- **MariaDB `Specified key was too long`** — the `99-frappe.cnf` drop-in isn't being
  read. Check `sudo mariadb -e "SHOW VARIABLES LIKE 'character_set_server';"` shows
  `utf8mb4`, then `sudo systemctl restart mariadb`.
- **`nginx: [emerg] unknown log format "main"`** — add the `main` format per
  [Configure the edge and Nginx](#10-configure-the-edge-and-nginx).
- **Cloudflare 521 (Web server is down)** — the origin refused the connection. Either
  Nginx isn't running, or the security group is blocking the current Cloudflare
  ranges on 443 (80 in Flexible mode).
- **Cloudflare 525 (SSL handshake failed)** — the Origin Cert isn't installed or
  presented correctly. Run `sudo nginx -t`, confirm
  `/etc/ssl/cloudflare/origin.{pem,key}` exist with `644`/`600` permissions, the
  `wildcard` block points at the right paths, and `bench setup nginx --yes` ran after
  you edited it.

## References

- [Setup production](https://docs.frappe.io/framework/v15/user/en/bench/guides/setup-production)
- [Adding custom domains](https://docs.frappe.io/framework/user/en/bench/guides/adding-custom-domains)
- [Site configuration](https://docs.frappe.io/framework/user/en/basics/site_config)
- [Cloudflare origin CA](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/)
- [Cloudflare IP ranges (IPv4)](https://www.cloudflare.com/ips-v4),
  [(IPv6)](https://www.cloudflare.com/ips-v6)
- [ossutil](https://www.alibabacloud.com/help/en/oss/developer-reference/ossutil-overview/)
- [Node version manager](https://github.com/nvm-sh/nvm)
