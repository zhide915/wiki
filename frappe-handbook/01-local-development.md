# Local Development

Set up a Frappe **V15** development bench on Windows under WSL Ubuntu, ending with a
site running at `http://localhost:8000`. This guide assumes a Windows machine on
which you can run PowerShell as Administrator to install WSL; everything after that
runs inside WSL Ubuntu. See the [conventions and placeholders](README.md#conventions).

This is the single happy path: Frappe V15 on Python 3.11 and Node 18. For another
major, swap the versions from the matrix below — the steps are otherwise identical.
`uv` manages Python and `nvm` manages Node, so several majors live side by side on
one machine.

|         | Frappe V14 | Frappe V15 | Frappe V16 |
| ------- | ---------- | ---------- | ---------- |
| Python  | 3.11       | 3.11       | 3.14       |
| Node.js | 18         | 18         | 24         |

## Contents

1. [Prepare the environment](#1-prepare-the-environment)
2. [Install MariaDB](#2-install-mariadb)
3. [Install Redis and wkhtmltopdf](#3-install-redis-and-wkhtmltopdf)
4. [Install the toolchain](#4-install-the-toolchain)
5. [Initialize the bench](#5-initialize-the-bench)
6. [Create a site and run](#6-create-a-site-and-run)
7. [Troubleshooting](#troubleshooting)

## 1. Prepare the environment

Install WSL Ubuntu, then the build packages every later step depends on.

From **PowerShell as Administrator**, install the WSL Ubuntu distribution:

```powershell
wsl --install -d Ubuntu
```

Open the Ubuntu terminal and choose a username and password. Run everything from
here on **inside WSL**, not PowerShell. Refresh the package index and upgrade
installed packages first:

```bash
cd ~
sudo apt update
sudo apt upgrade -y
```

Install the build dependencies — the C toolchain, the headers Python C-extensions
compile against, and the X and font libraries wkhtmltopdf needs:

```bash
sudo apt install -y \
  git curl wget build-essential \
  pkg-config libssl-dev libffi-dev \
  zlib1g-dev software-properties-common \
  xvfb libfontconfig xfonts-75dpi
```

## 2. Install MariaDB

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
— keep it, because you need it for every `bench new-site`:

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

bind-address = 0.0.0.0

[mysql]
default-character-set = utf8mb4

[mysqldump]
default-character-set = utf8mb4
EOF

sudo systemctl restart mariadb
```

- The `utf8mb4` block is required. Without it, `bench new-site` fails with
  `Specified key was too long`.
- `innodb-buffer-pool-size = 2G` is InnoDB's RAM cache; scale it to your machine.
- `bind-address = 0.0.0.0` opens MariaDB to Windows tools reaching into WSL. Set it
  to `127.0.0.1` if you don't want that. ⚠️ Never use `0.0.0.0` on a server.

### Optional: create a remote user

Do this only to connect from a Windows tool such as MySQL Workbench into WSL. Open a
root MariaDB shell:

```bash
sudo mariadb -u root -p
```

Create a user that can connect from any host and grant it full privileges:

```sql
CREATE USER 'remote'@'%' IDENTIFIED BY 'YOUR_REMOTE_PW';
GRANT ALL PRIVILEGES ON *.* TO 'remote'@'%';
FLUSH PRIVILEGES;
```

> ⚠️ `'%'` accepts connections from any IP, and `ALL PRIVILEGES ON *.*` hands over
> full access. This is fine inside WSL on a development machine; never do it on a
> server.

## 3. Install Redis and wkhtmltopdf

Redis backs Frappe's cache, queues, and real-time messaging. The defaults are fine
locally:

```bash
sudo apt install -y redis-server
```

Frappe renders PDFs with the **patched-Qt** build of wkhtmltopdf; the distribution
package does not work. Install the prebuilt `.deb` and check its version:

```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_amd64.deb
sudo dpkg -i wkhtmltox_0.12.6.1-3.jammy_amd64.deb
sudo apt install -f -y
wkhtmltopdf --version
```

wkhtmltopdf is the correct build when the version output contains `with patched qt`.
If it does not, PDF generation fails at runtime — see [Troubleshooting](#troubleshooting).

## 4. Install the toolchain

`uv` manages Python and `nvm` manages Node, so multiple Frappe majors coexist
without touching the system Python.

Install `uv`, reload your shell so the new `PATH` takes effect, then install Python
3.11:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv python install 3.11
```

Install `nvm`, reload your shell, then install Node 18 and Yarn:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.5/install.sh | bash
source ~/.bashrc
nvm install 18
npm install -g yarn
```

Install Bench and pre-commit as global `uv` tools. Installing Bench this way keeps
it global without leaking into any bench's environment:

```bash
uv tool install frappe-bench
uv tool install pre-commit
bench --version
```

Turn on pre-commit inside an app repository with `pre-commit install` when you start
work there. The toolchain is ready when `bench --version` prints a version rather
than `command not found`.

## 5. Initialize the bench

A *bench* is a self-contained Frappe project: its own Python virtualenv, its own
`node_modules`, and one or more sites under it.

Create the parent directory, select Node 18, and initialize the bench on the V15
branch with the Python 3.11 interpreter:

```bash
mkdir -p ~/frappe
cd ~/frappe
nvm use 18
bench init frappe-bench --frappe-branch version-15 --python $(uv python find 3.11)
```

This takes a few minutes: it clones Frappe, builds the virtualenv, and compiles
assets. The result is the bench directory tree:

```
~/frappe/frappe-bench/
├── apps/frappe/        # the Frappe framework
│   └── node_modules/   # Node deps live per app, not at the bench root
├── env/                # Python virtualenv (isolated)
└── sites/              # sites get created here
```

Everything below runs from inside the bench. Pin `setuptools<81` first: the
transitive dependency `googleapiclient` still imports `pkg_resources`, which was
removed in setuptools 81+. Then enable developer mode and server scripts:

```bash
cd <bench>
./env/bin/pip install "setuptools<81"
bench set-config -g allow_tests 1
bench set-config -g developer_mode 1
bench set-config -g server_script_enabled 1
```

Developer mode lets you create and edit DocTypes; server scripts let you author
server-side Python from the Frappe UI.

### Optional: fetch ERPNext or HRMS

You can develop against bare Frappe alone. To build on ERPNext or HRMS, pull them
onto the bench now — they are installed on a site in [step 6](#6-create-a-site-and-run):

```bash
bench get-app --branch version-15 erpnext
bench get-app --branch version-15 hrms
```

### Optional: auto-activate the bench with direnv

[direnv](https://direnv.net/) activates the virtualenv and selects Node the moment
you `cd` into the bench, with no manual `source env/bin/activate` or `nvm use`.

Install direnv and hook it into your shell. The shell hook is what makes direnv run;
use the matching line (`direnv hook zsh`, and so on) if your shell isn't bash:

```bash
sudo apt install -y direnv
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc
```

Add two shared helpers to your direnv config — `use_uv` activates a virtualenv and
`use_nvm` selects a Node version:

```bash
mkdir -p ~/.config/direnv
cat > ~/.config/direnv/direnvrc <<'EOF'
use_uv() {
  local venv_dir=${1:-.venv}
  if [[ -e "$venv_dir/bin/activate" ]]; then
    source "$venv_dir/bin/activate"
  fi
}

use_nvm() {
  local node_version=$1
  nvm_sh="$HOME/.nvm/nvm.sh"
  if [[ -e "$nvm_sh" ]]; then
    source "$nvm_sh"
    nvm use "$node_version"
  fi
}
EOF
```

Add an `.envrc` in the bench that calls the helpers, then authorize it. direnv
ignores an `.envrc` until you run `direnv allow`, and re-prompts whenever the file
changes:

```bash
cd <bench>
cat > .envrc <<'EOF'
use_uv env
use_nvm 18
EOF

direnv allow
```

The `direnvrc` helpers are shared, so for each new bench you add only the `.envrc`
and run `direnv allow`.

## 6. Create a site and run

Inside the bench, a *site* is one database-backed Frappe instance.

Create the site. The `.localhost` suffix resolves to `127.0.0.1` in every modern
browser, so the site works with no `hosts`-file edits:

```bash
cd <bench>
bench new-site mysite.localhost
```

You are asked for the MariaDB root password (from [step 2](#2-install-mariadb)) and
a new Administrator password. Save both.

If you fetched ERPNext or HRMS, install it on the site now. Install any dependency
first — HRMS needs ERPNext:

```bash
bench --site mysite.localhost install-app erpnext
```

Make the site the default, then start the development server:

```bash
bench use mysite.localhost
bench start
```

The site is live when [http://localhost:8000](http://localhost:8000) loads and you
can sign in as **Administrator**. Stop the server with `Ctrl+C`.

## Troubleshooting

- **`bench new-site` fails: "Access denied for user 'root'@'localhost'"** — root is
  still on `unix_socket` auth. Re-run the `ALTER USER` from
  [step 2](#2-install-mariadb).
- **`ModuleNotFoundError: No module named 'pkg_resources'`** — pin setuptools in the
  bench env: from inside `<bench>`, run `./env/bin/pip install "setuptools<81"`.
- **`bench --version` says "command not found"** — the new `PATH` hasn't reached
  this shell. Run `source ~/.bashrc` or reopen the terminal.
- **`wkhtmltopdf --version` doesn't show "with patched qt"** — the wrong build is
  installed. Run `sudo apt remove wkhtmltopdf`, then redo
  [step 3](#3-install-redis-and-wkhtmltopdf).
- **`bench init` fails partway** — usually a missing build dependency; confirm
  [step 1](#1-prepare-the-environment) finished in full. If the error is
  Python-related, check that you passed the right interpreter
  (`uv python find 3.11`).
- **Browser shows "Site does not exist"** — you ran `bench start` without
  `bench use mysite.localhost`. Set the default site, or visit
  `http://mysite.localhost:8000` directly.

## References

- [Bench commands](https://docs.frappe.io/framework/v15/user/en/bench/bench-commands)
- [Migrating to version 15](https://github.com/frappe/frappe/wiki/Migrating-to-version-15)
- [uv](https://docs.astral.sh/uv/)
- [Node version manager](https://github.com/nvm-sh/nvm)
- [wkhtmltopdf releases](https://github.com/wkhtmltopdf/packaging/releases)
