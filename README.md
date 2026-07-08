# Ansible + Semaphore: an Enterprise-Ready Lab

This is a working Ansible project plus a guide for running it through
[Semaphore](https://semaphoreui.com), Ansible's open-source UI/orchestrator.
It's built around a concrete scenario:

- **Control node**: one Ubuntu host, running Ansible and Semaphore.
- **Managed nodes**: one Ubuntu host, one openSUSE host, one RHEL (or
  RHEL-family: Rocky, AlmaLinux, CentOS Stream) host.
- **Automation goals**:
  1. Provision human admin accounts that authenticate over SSH with keys.
  2. Deploy a simple nginx webserver to all three hosts, or any subset.

Everything in `inventory/`, `playbooks/`, and `roles/` is real, runnable
code -- not pseudo-code. It has been syntax-checked, linted against
ansible-lint's `production` profile, and exercised end-to-end (account
creation, key deployment, sudo policy, idempotency, and an offboarding
scenario) before being handed to you.

```
.
├── ansible.cfg
├── requirements.txt              # control-node Python deps
├── requirements.yml              # Ansible Galaxy collections
├── inventory/production/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all/
│       │   ├── vars.yml          # shared, non-secret variables
│       │   └── vault.yml         # ansible-vault encrypted example
│       ├── ubuntu.yml
│       ├── opensuse.yml
│       └── redhat.yml
├── playbooks/
│   ├── bootstrap_service_account.yml   # run once, manually
│   ├── users.yml                       # day-2, idempotent
│   ├── webserver.yml                   # day-2, idempotent
│   └── site.yml                        # convenience: both of the above
├── roles/
│   ├── admin_users/
│   └── webserver/
└── semaphore/
    ├── docker-compose.yml
    └── .env.example
```

---

## 1. Architecture and the order things happen in

A detail that trips up a lot of first-time enterprise Ansible setups:
**you can't manage a host with Ansible until something already lets you
log into it.** That "something" is usually a cloud-init default user, or
root with a console password. The right pattern is:

1. **Bootstrap** (manual, once per host, higher privilege): use whatever
   initial access you have to create one dedicated, non-interactive
   **service account** (`ansible_svc` in this repo) and authorize
   Semaphore's automation key for it. Then retire the bootstrap
   credential (rotate the root password, disable the cloud-init user --
   whatever your platform calls for).
2. **Day 2** (automated, repeatable, idempotent, runs *as* `ansible_svc`):
   everything else -- provisioning human admins, deploying the webserver,
   any future playbook -- runs through Semaphore using that service
   account.

This is why `playbooks/bootstrap_service_account.yml` is deliberately kept
separate from `playbooks/users.yml`. They have different blast radii,
different credentials, and different cadences (run once vs. run
constantly/scheduled). Mixing them into one "do everything" playbook is a
common anti-pattern -- resist the urge to merge them even though it would
save a step.

Why a dedicated service account instead of just using root or the
cloud-init user forever:

- **Least privilege in spirit, auditability in practice.** Ansible modules
  ultimately need to run arbitrary privileged commands, so you can't
  meaningfully restrict *what* the account can do via sudoers without
  breaking modules in subtle ways. What you *can* do is restrict the
  account to one purpose, give its private key to nothing except
  Semaphore's encrypted key store, and rely on Semaphore's task history as
  your audit log of every command that account ever ran.
- A compromised/rotated automation key doesn't affect human logins, and
  vice versa.
- It survives the human admin list changing -- no human's offboarding can
  ever accidentally take down the automation pipeline.

---

## 2. Installing Ansible on the Ubuntu control node

**Don't use `apt install ansible`.** Ubuntu's repo version lags upstream
significantly, and you lose the ability to pin a specific Ansible
version per project -- which matters once you have more than one
project on the same control node. Use an isolated Python environment
instead:

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip git

git clone <your-git-remote-for-this-repo> /opt/ansible-semaphore-enterprise-lab
cd /opt/ansible-semaphore-enterprise-lab

python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt

ansible-galaxy collection install -r requirements.yml
```

`requirements.txt` pins `ansible-core` to a range (`>=2.16,<2.18`) rather
than to "latest" -- enterprise projects should control exactly when they
pick up a new Ansible version, the same way they would for any other
dependency. `requirements.yml` does the same for the two collections the
roles use (`ansible.posix`, `community.general`).

Verify it worked:

```bash
ansible --version
ansible-inventory -i inventory/production/hosts.yml --graph
```

The second command should print:

```
@all:
  |--@ungrouped:
  |--@managed_nodes:
  |  |--@ubuntu:
  |  |  |--ubuntu-node
  |  |--@opensuse:
  |  |  |--opensuse-node
  |  |--@redhat:
  |  |  |--redhat-node
  |--@webservers:
  |  |--@ubuntu:
  |  |  |--ubuntu-node
  |  |--@opensuse:
  |  |  |--opensuse-node
  |  |--@redhat:
  |  |  |--redhat-node
```

Before going further, edit `inventory/production/hosts.yml` and replace
the placeholder `ansible_host` IPs with your real ones.

### Why two groups containing the same three hosts?

`managed_nodes` is the "everything" group used for account management --
every host should get admin users. `webservers` is a *function* group
used for the webserver playbook. Right now they happen to contain the
same three hosts, but this structure is what lets you scale later: add a
fourth host that's a database server but not a webserver, and it joins
`managed_nodes` without ever joining `webservers`. A host can belong to
any number of groups -- model your inventory by *function*, not just by
*OS*, from day one.

---

## 3. Inventory and variable layout (the part people get wrong)

The three per-OS files under `inventory/production/group_vars/` are the
core of why the roles themselves don't need a single `when:
ansible_os_family == ...` for anything except firewall handling:

| Group      | `admin_users_sudo_group` | `webserver_docroot`     | `webserver_firewall_backend` |
|------------|---------------------------|--------------------------|-------------------------------|
| `ubuntu`   | `sudo`                    | `/var/www/html`         | `ufw`                         |
| `opensuse` | `wheel`*                  | `/srv/www/htdocs`*      | `firewalld`                   |
| `redhat`   | `wheel`                   | `/usr/share/nginx/html` | `firewalld`                   |

\* **Verify these two on your actual openSUSE image before running
anything.** Sudo group conventions and nginx's default doc root have
genuinely varied across openSUSE versions/spins. The package install
itself (`ansible.builtin.package`, name `nginx`) and the service
management (`ansible.builtin.service`, name `nginx`) are consistent
across all three distros, which is what lets the role avoid per-OS task
branching for those steps -- but doc root and sudo group are
configuration, not code, so they live in `group_vars`, not in the role.

This is the pattern to copy for any future per-OS difference you hit:
**push the difference into `group_vars`, not into `when:` conditions
inside roles**, whenever the difference is "what value" rather than
"what steps." Reserve `when: ansible_os_family == ...` branching for
genuine differences in *behavior* (which is exactly what the firewall
handling in `roles/webserver/tasks/main.yml` does, since `ufw` and
`firewalld` are entirely different modules, not just different values of
one variable).

`group_vars/all/vars.yml` holds variables identical across every host
(the admin user list, shared webserver defaults). Anything that varies
by OS family overrides it via the per-group files above -- standard
Ansible variable precedence (host_vars > group_vars on specific groups >
group_vars/all) does the rest.

---

## 4. Secrets: ansible-vault vs. Semaphore Environments

You have two legitimate places to put a secret in this stack, and they
solve different problems:

- **`ansible-vault`** (`inventory/production/group_vars/all/vault.yml` in
  this repo) is for a secret that needs to travel *with the code* --
  something a playbook reads as a normal Ansible variable, that must be
  reviewable in git history and consistent across however many places run
  this repo. It's encrypted at rest in git, but anyone with the vault
  password and the repo can decrypt it locally.
- **Semaphore Environments** (a first-class object in Semaphore, separate
  from the git repo) are for secrets that are specific to *where this is
  deployed from* -- the kind of thing you don't want sitting in git at
  all, even encrypted, because it's bound to one Semaphore instance's
  encrypted-at-rest database rather than your whole engineering org's git
  history. API tokens, webhook URLs, anything you'd want to revoke
  without touching the repo.

A reasonable default split: **vault for anything the playbook logic
depends on; Semaphore Environment variables for anything operational**
(notification webhooks, per-environment toggles). This repo includes a
real, working vault example so you can see the actual encrypted file
format:

```bash
ansible-vault view inventory/production/group_vars/all/vault.yml
# decrypts the example vault_breakglass_password_hash variable
```

To make your own:

```bash
ansible-vault encrypt_string 'super-secret-value' --name 'my_secret_var'
# paste the output into a vars file, or
ansible-vault edit inventory/production/group_vars/all/vault.yml
```

Store the vault password itself in Semaphore as a **Vault Password**-type
key (Key Store → New Key → type "Vault Password") attached to the Task
Templates that need it -- never as a file sitting on the control node's
disk next to the repo it decrypts.

---

## 5. SSH host keys (don't just turn off host key checking)

`ansible.cfg` in this repo sets `host_key_checking = True` deliberately.
The common shortcut -- `host_key_checking = False`, or
`ANSIBLE_HOST_KEY_CHECKING=False` as an env var -- silently removes your
only protection against connecting to the wrong (or a spoofed) host. The
reason people reach for it is that an *interactive* "are you sure you
want to continue connecting?" prompt has no TTY to answer when Semaphore
runs a job, so it just hangs.

The fix that keeps the protection: point SSH at an explicit,
repo-tracked `known_hosts` file instead of disabling checking entirely.
`ansible.cfg`'s `ssh_args` already does this
(`-o UserKnownHostsFile=./.known_hosts`). Populate it once per host, from
a channel you trust more than a blind network round-trip -- your cloud
provider's console output, your provisioning system's output, or at
minimum `ssh-keyscan` run from the control node itself right after first
boot:

```bash
ssh-keyscan -t ed25519 ubuntu-node-ip opensuse-node-ip redhat-node-ip >> .known_hosts
```

Commit `.known_hosts` to the repo (it's not noted in `.gitignore` --
that's intentional; it's the opposite of a secret, it's the thing
protecting you against one).

---

## 6. The `admin_users` role

`playbooks/users.yml` runs the `admin_users` role against
`managed_nodes`. The single source of truth is
`admin_users_accounts` in `inventory/production/group_vars/all/vars.yml`:

```yaml
admin_users_accounts:
  - username: alice
    comment: "Alice Admin"
    ssh_pubkey: "ssh-ed25519 AAAA... alice@example.com"
    sudo: true
```

What it does, per entry:

- Creates the account (locked password -- `password: "!"` -- so SSH key
  is the *only* way in; there's no password to brute-force or phish).
- Adds it to the OS-appropriate sudo group, only if `sudo: true`.
- Deploys the public key with `exclusive: true`, so the file always
  matches exactly what's declared in `admin_users_accounts` -- rotate a
  key by changing `ssh_pubkey` and re-running; the old key is gone.
- Renders one sudoers drop-in for the whole admin group, validated with
  `visudo -cf` *before* it's written -- a syntactically broken sudoers
  file written without validation is one of the more reliable ways to
  lock yourself out of a box.

### Offboarding: read this before you delete anyone

**Do not delete someone's entry from `admin_users_accounts` to remove
their access.** This was tested explicitly while building this repo: an
Ansible role only acts on what's listed, so deleting the entry just
means Ansible stops looking at that user -- their account, sudo
membership, and SSH key all stay exactly as they were.

To actually revoke access, flip their entry's `state` to `absent` and
**keep the entry**:

```yaml
  - username: carol
    state: absent
```

That run will: delete `carol`'s `authorized_keys`, strip her sudo group
membership, and set her shell to `/usr/sbin/nologin`. The account and
home directory are kept (useful for forensics / audit trails) unless you
also set `admin_users_purge_absent: true`, which makes that last step a
full `userdel --remove`.

### Should sudo need a password?

`admin_users_sudo_requires_password: true` (the default in
`group_vars/all/vars.yml`) means human admins type their own password for
`sudo`, even though they logged in with a key. That's the right default
for anything beyond a personal sandbox -- it ensures a stolen, unlocked
laptop session isn't an automatic root shell. Flip it to `false` only if
you have a specific, considered reason to.

---

## 7. The `webserver` role

`playbooks/webserver.yml` targets the `webservers` group. Run it against
everyone:

```bash
ansible-playbook playbooks/webserver.yml
```

...or against a subset, two equivalent ways depending on whether you
think in terms of OS or individual hosts:

```bash
ansible-playbook playbooks/webserver.yml --limit ubuntu
ansible-playbook playbooks/webserver.yml --limit 'redhat,opensuse'
ansible-playbook playbooks/webserver.yml --limit ubuntu-node
```

What it does:

1. `ansible.builtin.package: name=nginx` -- the generic module, not
   `apt`/`dnf`/`zypper`-specific ones. Ansible inspects the target host's
   facts and picks the right backend automatically. This is the detail
   that lets one task work across three different distros with zero
   `when:` conditions.
2. `ansible.builtin.service` enables and starts it -- same logic, one
   module, auto-detected backend (systemd on all three targets here).
3. Templates `index.html.j2` into `webserver_docroot` (the per-OS path
   from `group_vars`), showing the host's own facts -- distro, OS family,
   kernel, deploy timestamp -- so you can visibly confirm which host
   served which page.
4. Opens the firewall, but *how* genuinely differs by OS, so this is
   exactly the kind of difference that belongs in `when:` branching
   rather than a variable: `community.general.ufw` on Debian-family
   hosts, `ansible.posix.firewalld` on RHEL/SUSE-family hosts. Notice the
   ufw branch explicitly allows OpenSSH *before* it touches any other
   rule -- `ufw` defaults to deny-by-default once enabled, and the
   single fastest way to lock yourself out of a remote box is to open a
   port without first confirming SSH survives the same change.

A content change (re-running with a different template) triggers the
`Reload webserver` handler rather than a full restart -- a config/content
reload is enough here and avoids a brief connection drop a restart would
cause.

---

## 8. Setting up Semaphore

Semaphore is the orchestration layer: it stores your Ansible project,
credentials, and inventories, runs playbooks on a schedule or on demand,
shows real-time output, and keeps an audit trail of who ran what and
when. It runs **on the same Ubuntu control node** as the Ansible
installation above (it bundles its own Ansible, but pointing it at the
same git repo means what you tested by hand and what Semaphore runs are
guaranteed to be identical).

### 8.1 Install via Docker Compose with a Postgres backend

`semaphore/docker-compose.yml` in this repo does this. For anything
beyond a personal sandbox, prefer Postgres (or MySQL) over Semaphore's
embedded BoltDB -- you get proper backups, replication options, and the
ability to point a second Semaphore instance at the same DB for HA later.

```bash
sudo apt install -y docker.io docker-compose-v2
cd semaphore
cp .env.example .env
```

Generate the three encryption secrets and fill in `.env`:

```bash
openssl rand -base64 32   # -> SEMAPHORE_COOKIE_HASH
openssl rand -base64 32   # -> SEMAPHORE_COOKIE_ENCRYPTION
openssl rand -base64 32   # -> SEMAPHORE_ACCESS_KEY_ENCRYPTION
```

These three encrypt everything Semaphore stores at rest -- every saved
SSH key, vault password, and login credential. Treat them like a root CA
private key: back them up somewhere durable and access-controlled, and
understand that losing them means losing access to every secret
Semaphore is holding (not just inconvenience -- actual data loss for
those credentials).

Also set `SEMAPHORE_DB_PASS` and `SEMAPHORE_ADMIN_PASS` to real random
values, then:

```bash
docker compose up -d
docker compose logs -f semaphore   # watch it come up
```

By default the container only binds to `127.0.0.1:3000` -- it is not
reachable from outside this host yet. That's deliberate; see 8.2.

### 8.2 Put it behind TLS

Don't expose Semaphore's HTTP port directly to the internet -- you're
about to put SSH private keys and sudo-capable credentials into it.
Terminate TLS in front of it with a reverse proxy. Caddy is the lowest-
effort option (automatic Let's Encrypt certs):

```bash
sudo apt install -y caddy
```

```
# /etc/caddy/Caddyfile
semaphore.example.com {
    reverse_proxy 127.0.0.1:3000
}
```

```bash
sudo systemctl reload caddy
```

(nginx + certbot or Traefik work just as well if that's what your
organization already standardizes on.)

### 8.3 First login and the object model

Log in at `https://semaphore.example.com` with the admin credentials from
`.env`. Semaphore's data model, in the order you'll set them up:

1. **Project** -- create one, e.g. "Enterprise Lab".
2. **Key Store** -- this is where credentials live, encrypted, and never
   touch the control node's disk as plaintext files:
   - An **SSH key** entry holding the `ansible_svc` private key (the
     counterpart to the public key you authorized in step 9 below).
   - A **Vault Password** entry, if you're using `ansible-vault` (see
     section 4) -- the password for
     `inventory/production/group_vars/all/vault.yml`.
3. **Repository** -- point it at this project's git remote and branch.
4. **Inventory** -- choose "File" and point it at
   `inventory/production/hosts.yml` from the repo, rather than re-typing
   hosts into Semaphore's UI. This keeps one source of truth: edit the
   repo, not the UI, when hosts change.
5. **Environment** -- holds extra vars and/or secret variables specific
   to this Semaphore instance (section 4). Can be empty to start.
6. **Task Template** -- the thing you actually click "Run" on. One per
   playbook is the right granularity here (see 8.4), each referencing the
   Repository, Inventory, the `ansible_svc` SSH key, and (for
   `users.yml`) the Vault Password key.

Exact field names/layout shift slightly between Semaphore releases --
this describes the model as of recent 2.x versions; if something doesn't
match your screen 1:1, the official docs
(<https://docs.semaphoreui.com>) are the source of truth.

### 8.4 One Task Template per playbook, not one for everything

Create separate templates for `playbooks/users.yml` and
`playbooks/webserver.yml`, rather than one template running
`playbooks/site.yml`. This isn't just tidiness:

- They have different natural cadences -- you might schedule `users.yml`
  nightly to catch drift, while `webserver.yml` is closer to "run when I
  change the site."
- They can have different RBAC: in Semaphore's Teams/permissions model,
  you can restrict who's allowed to run which template. Account
  provisioning and webserver deploys plausibly warrant different
  approvers.
- A failed/partial run has a smaller, clearer blast radius when each
  template does one thing.

`playbooks/site.yml` is still in the repo for the rare case you
genuinely want "run both, in order" as a single button -- just don't make
it your default.

### 8.5 Letting an operator pick a subset of hosts from the UI

Add a **Survey Variable** on the `webserver.yml` template (Task Template
→ Survey Variables) named something like `target_hosts`, type "String",
with a sensible default like `webservers`. Then set the template's
**CLI Args** field to:

```
--limit {{ target_hosts }}
```

Now running the template from the Semaphore UI prompts for which
hosts/group to target, instead of requiring someone to edit YAML or use
the CLI directly. (Exact survey-variable mechanics -- whether it's
injected as an extra-var or substituted directly into CLI args -- vary
slightly by Semaphore version; check the Task Template's help text in
your installed version.)

### 8.6 Audit trail

Every run shows up in Semaphore's Task History with full output, who
triggered it, and when. This is, in practice, your primary justification
for the `ansible_svc` / NOPASSWD-sudo trade-off discussed in section 1:
the account itself is broad, but every single thing it ever does is
logged centrally and isn't something a human can quietly do off the
record.

---

## 9. Bootstrapping the three managed hosts

Generate a dedicated keypair for Semaphore -- don't reuse a personal SSH
key:

```bash
ssh-keygen -t ed25519 -f semaphore_automation_key -C "semaphore-ansible-svc" -N ""
```

Run the bootstrap playbook against each host using whatever initial
access you currently have. Example for cloud-init-style Ubuntu/RHEL/
openSUSE images where you can SSH in as a sudo-capable default user:

```bash
ansible-playbook -i inventory/production/hosts.yml \
   --ask-become-pass --ask-vault-password \
  -e "{\"semaphore_pubkey\": \"$(cat semaphore_automation_key.pub)\"}" \
  playbooks/bootstrap_service_account.yml
```

Then:

1. Paste the **private** key (`semaphore_automation_key`) into Semaphore's
   Key Store as an SSH key credential, then delete the local copy of the
   private key file (`shred -u semaphore_automation_key` or equivalent) --
   Semaphore's encrypted store is now the only place it needs to exist.
2. Run the `users.yml` Task Template from Semaphore and confirm it
   succeeds as `ansible_svc`.
3. Retire the bootstrap credential per your platform's standard (disable
   the cloud-init user, rotate the root password, etc.).

---

## 10. Day-to-day operations

```bash
# Add/rotate/offboard a human admin: edit admin_users_accounts in
# inventory/production/group_vars/all/vars.yml, then run (or click "Run"
# on the users.yml Task Template in Semaphore)
ansible-playbook playbooks/users.yml

# Deploy/update the webserver everywhere
ansible-playbook playbooks/webserver.yml

# ...or on just one OS family
ansible-playbook playbooks/webserver.yml --limit opensuse

# Preview changes without applying them
ansible-playbook playbooks/webserver.yml --check --diff

# Lint before you commit -- this repo passes the strict "production" profile
ansible-lint playbooks/ roles/
```

---

## 11. Where to go from here

This repo deliberately stops at "two solid playbooks done right," not
"every enterprise feature." Reasonable next steps, roughly in the order
most teams reach for them:

- **SSH hardening** (`PasswordAuthentication no`, `PermitRootLogin no` in
  `sshd_config`) -- genuinely high value, but risky to apply blind: make
  absolutely sure at least one key-based admin account already works
  *before* disabling password auth, or you can lock yourself out
  permanently. Worth its own carefully-tested playbook rather than
  bolting it onto this one.
- **Molecule** for testing roles against real containers/VMs in CI,
  rather than relying on `--syntax-check` and manual runs.
- **LDAP/OIDC integration** in Semaphore, once you have more than a
  handful of operators, instead of Semaphore-local accounts.
- **A second Semaphore instance against the same Postgres DB** for HA,
  once this stops being a lab.
- **Scheduled drift-detection runs** of `users.yml` (Semaphore supports
  cron-style schedules on Task Templates) so configuration drift gets
  caught and corrected automatically, not just on demand.
