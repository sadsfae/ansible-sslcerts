# ansible-sslcerts
Simple Ansible playbook to replace Apache and Nginx SSL certificates

[![GA](https://github.com/sadsfae/ansible-sslcerts/actions/workflows/ansible-lint.yml/badge.svg)](https://github.com/sadsfae/ansible-sslcerts/actions)

* This takes SSL certificates and keys copied into `install/roles/sslcerts/files` and copies them to servers that match their name.
  - e.g. `host-01.pem` (certificate) and `host-01.key` (certificate key)
* This then restarts each respective webserver type you are using.

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

#### Run the Playbook

* Run the playbook:
```
ansible-playbook -i hosts install/sslcerts.yml
```

### Notes
* Make sure you modify your web server(s) to expect these filenames based on their FQDN or domain.
* You can easily generate your own TLS certificate/key with one command for testing:

```
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 3650 -nodes -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=CommonNameOrHostname"
```

### To Do
* Inspect the local certificates to make sure they match the domain
