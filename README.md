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
* By default every host needs its own `<hostname>.pem` and `<hostname>.key` in `install/roles/sslcerts/files`, and the playbook expects those files to exist.
* If you would rather serve one wildcard certificate to hosts that do not have a host-specific file, set `use_seed_file_if_absent: true` in `install/group_vars/all.yml` (it defaults to `false`, so behavior is unchanged unless you opt in).
* Place the wildcard certificate and key on the Ansible control node (the machine that runs the playbook), relative to `install/roles/sslcerts/files/`, and point the globals at them:

```yaml
use_seed_file_if_absent: true
wildcard_cert_file: wildcard.pem
wildcard_key_file: wildcard.key
```

* With this enabled, the role checks the control node for `install/roles/sslcerts/files/<hostname>.pem` and `<hostname>.key` while running (`delegate_to: localhost`). If either file is missing, it is seeded from the files named by `wildcard_cert_file` / `wildcard_key_file` (created with `mode: '0600'`).
* Host-specific files always win: seeding only happens when the per-host file is absent, so an explicitly placed certificate is never overwritten. The seeded files are then pushed to the server exactly like an explicit one.
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
