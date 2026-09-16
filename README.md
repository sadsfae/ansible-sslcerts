# ansible-sslcerts
Simple Ansible playbook to replace Apache and Nginx SSL certificates

[![GA](https://github.com/sadsfae/ansible-sslcerts/actions/workflows/ansible-lint.yml/badge.svg)](https://github.com/sadsfae/ansible-sslcerts/actions)

* This takes SSL certificates and keys copied into `install/roles/sslcerts/files` and copies them to servers that match their name.
  - e.g. `host-01.pem` (certificate) and `host-01.key` (certificate key)
* This then restarts each respective webserver type you are using.

## Table of Contents

- [Apache and Nginx Files Locations](#apache-and-nginx-files-locations)
- [How to Use](#how-to-use)
  - [Edit Hosts File](#edit-hosts-file)
  - [Generate New Certificates](#generate-new-certificates)
  - [Using a Wildcard Certificate](#using-a-wildcard-certificate)
  - [Host-level Ownership and Mode Overrides](#host-level-ownership-and-mode-overrides)
  - [Rootless Container (Podman)](#rootless-container-podman)
  - [Run the Playbook](#run-the-playbook)
- [Notes](#notes)
- [To Do](#to-do)

## Apache and Nginx Files Locations

| SSL Component | File System Path                    |
| --------------|-------------------------------------|
| Apache Cert   | /etc/pki/tls/certs/servername.pem   |
| Apache Key    | /etc/pki/tls/private/servername.key |
| Nginx Cert    | /etc/pki/tls/certs/servername.pem   |
| Nginx Key     | /etc/pki/tls/certs/servername.key   |

* You can change this to your liking in `install/group_vars/all.yml`

## How to Use
#### Edit Hosts File
* Edit the `hosts` inventory as follows, depending on nginx or Apache

```
[apache]
host-01

[nginx]
host-02
```
#### Generate New Certificates
* Generate certificates and keys via your preferred method and name them appropriately.

```
install/roles/sslcerts/files
├── host-01.key
├── host-01.pem
├── host-02.key
└── host-02.pem
```

#### Using a Wildcard Certificate
* By default every host needs its own `<hostname>.pem` and `<hostname>.key` in `install/roles/sslcerts/files`, and a missing file fails the run loudly.
* If you would rather serve one wildcard certificate to hosts that do not have a host-specific file, set `use_seed_file_if_absent: true` in `install/roles/sslcerts/defaults/main.yml` (or override it anywhere Ansible picks up variables, e.g. `group_vars`). It defaults to `false`, so behavior is unchanged unless you opt in.
* Place the wildcard certificate and key on the Ansible control node (the machine that runs the playbook), relative to `install/roles/sslcerts/files/`, and point the defaults at them:

```yaml
use_seed_file_if_absent: true
wildcard_cert_file: wildcard.pem
wildcard_key_file: wildcard.key
```

* With this enabled, each copy task uses the per-host file when it exists and the wildcard file otherwise — the wildcard content is copied straight to the server. **Nothing is written into `files/`**, so no private key ever lands in the repo, and a renewed wildcard propagates automatically on the next run with no manual cleanup.
* Host-specific files always win: the fallback only applies when the per-host file is absent.
* Safety guards: a host that has only one of `<hostname>.pem` / `<hostname>.key` fails the run instead of silently deploying a mismatched pair, and the fallback is only considered for hosts in the `[apache]` or `[nginx]` groups.
* **Naming:** per-host files are matched against the system hostname (`ansible_nodename`, i.e. `uname -n`), not whatever you wrote in the inventory — name the files after the system hostname.
* Only turn this on if your wildcard actually covers the hosts you add: a `*.example.com` cert covers `host-01.example.com` but not `host-02.example.org`.

#### Host-level Ownership and Mode Overrides
* The copy tasks default to `owner: root`, `group: root`, `mode: '0600'` on the target hosts. If one host needs different attributes (for example, an application that has to read the private key through a shared group), override them per host without affecting the rest of the fleet.
* Create `install/host_vars/<fqdn>.yml` (fqdn = the host name as written in the `hosts` inventory) and set any of the following:

```yaml
sslcerts_owner: root
sslcerts_group: webapps
sslcerts_mode: '0640'
```

* Any value you leave out falls back to the default (`root` / `root` / `0600`). The overrides apply to both the Apache and Nginx copy tasks for that single host; other hosts keep the defaults. Set the group by name — Ansible resolves it to the GID on each target, so the same override works across hosts with different GIDs.
* `sslcerts_mode` applies to **both** the certificate and the key file, and per-host files are matched against the system hostname (`ansible_nodename`), not the inventory name.

#### Rootless Container (Podman)
* No new inventory group is needed: list the container host under the
  existing `[nginx]` group in your `hosts` inventory (that group triggers the
  nginx cert tasks). The same playbook command runs both kinds of host; a
  host's `host_vars` file decides whether it behaves like a container. The
  shipped `hosts` inventory shows this with a commented placeholder:

```ini
[nginx]
host-02
host-03
#host04        # rootless Podman nginx container host (see host_vars/host04.yaml)
```

* For nginx running in a rootless Podman container (systemd user Quadlet),
  override the cert destination, ownership and reload per host with
  `install/host_vars/<inventory_hostname>.yml` (the host name exactly as
  written in your `hosts` inventory). A shipped example is
  `install/host_vars/host04.yaml`:

```yaml
sslcerts_owner: qiip
sslcerts_group: qiip
sslcerts_mode: "0600"
nginx_cert_path: /home/qiip/.config/qiip-nginx/certs
nginx_key_path: /home/qiip/.config/qiip-nginx/certs
sslcerts_nginx_reload_command: systemctl --user --machine=qiip@.host restart qiip-nginx
```

* The cert dir must be the bind mount source the container mounts at
  `/etc/pki/tls/certs`, owned by the podman user: a rootless container cannot
  read root:root `0600` files.
* Quote `sslcerts_mode`: unquoted YAML `0600` is octal and renders as a
  decimal integer.
* `sslcerts_nginx_reload_command` is empty by default and restarts the system
  `nginx` service (historical behavior). When set, it runs instead, which is
  how a container's user unit gets restarted. The value must be a single
  command with arguments (`ansible.builtin.command`, no shell): pipes, `&&`
  and redirects are not supported. Prefer a restart over a reload:
  the container re-applies the SELinux `:Z` label to cert files written after
  the container started.
* Role files are still named after `ansible_nodename` (the system hostname);
  keep it equal to the FQDN the webserver expects.

#### Run the Playbook

* Run the playbook:
```
ansible-playbook -i hosts install/sslcerts.yml
```

### Notes
* Make sure you modify your web server(s) to expect these filenames based on their FQDN or domain.
* You can easily generate your own TLS certificate/key with one command for testing:

```
servername=$(hostname)
mkdir -p /etc/pki/tls/certs ; cd /etc/pki/tls/certs
openssl req -x509 -newkey rsa:4096 -keyout $servername.key -out $servername.pem -sha256 -days 3650 -nodes -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=CommonNameOrHostname"
```

### To Do
* Consider using Ansible Vault to store/manage certificate files
* Inspect the local certificates to make sure they match the target domains
