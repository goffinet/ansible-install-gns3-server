# Ansible Collection - Deploy GNS3 Server



This Ansible collection includes several roles to deploy GNS3 servers.

This code was tested with playbooks located in the `playbooks` folder. The Ansible Collection was not yet tested.

I use it to provide infrastructure labs through GNS3 to people I meet in my training classes.

Usualy, you can follow four steps to do it, as the four following playbooks :

1. `playbooks/provision.yml` : Provision Scaleway C2 instances (cheapest) or Packet servers (premium).
2. `playbooks/install-gns3-server.yml` : Install gns3-server stack with libvirtd/qemu/kvm, docker, fail2ban, openvpn, routing.
3. `playbooks/synchronize-gns3-files.yml` : Synchronize S3 bucket with default GNS3 folders on servers (AWS S3 object storage).
4. `playbooks/send-credits.yml` : Send credits to users. Everyone receives an email message telling him how to install an OpenVPN client, how to connect to the GNS3 server tunnel with the attached file, and how to install and configure the GNS3 client.

You can deploy or deprovision the solution in one step with those two playbooks :

* `deploy.yml` : This playbook read the four playbooks above.
* `deprovision.yml` : Deprovision (terminate) servers.

## Software requirements

Install `pip` and `git` :

- [How to install pip on Windows?](https://stackoverflow.com/questions/4750806/how-to-install-pip-on-windows)
- [How do I install pip on macOS or OS X?](https://stackoverflow.com/questions/17271319/how-do-i-install-pip-on-macos-or-os-x)
- [Proper way to install pip on Ubuntu](https://stackoverflow.com/questions/37954008/proper-way-to-install-pip-on-ubuntu)

Download this repo :

```bash
git clone https://github.com/goffinet/ansible-install-gns3-server

```

Install requirements :

```bash
cd ansible-install-gns3-server
pip install -r requirements.txt
```

## Inventory

Minimal (without any provision, sending mail and files synchronization) :

```ini
[gns3server]
gns3server0 ansible_host="172.16.31.0" mail_to="user0@domain.com"
gns3server1 ansible_host="172.16.31.1" mail_to="user1@domain.com"

[all:vars]
ansible_ssh_user=root
ansible_ssh_pass=testtest
ansible_port=22

```

Full (with all features) :

```ini
[gns3server]
gns3server0.mydomain.com mail_to="user0@domain.com" provider="scw" scw_type="C2S"
gns3server1.mydomain.com mail_to="user1@domain.com" provider="packet"

[all:vars]
zone="mydomain.com"
ansible_ssh_user=root
provider="scw" # packet
scw_type="C2M"
scw_region="par1"
scw_image_name="Ubuntu Bionic Beaver"
scw_image_arch="x86_64"
scw_image_size=10000000000
packet_facility="ams1"
packet_plan="t1.small.x86"
packet_os="ubuntu_18_04"
packet_project_id="88bd4dc8-8fe1-4130-8968-67147209365e"

```

## Secret credits

But you must also set secret variables in a file `playbooks/vars/secret.yml` as it :

```yaml
---
# Cloudflare DNS entries update
cloudflare_account_email: "mail@domain.com"
cloudflare_account_api_token: "XXXXX"
# Scaleway ID
scw_api_token: "XXXXX"
scw_organization: "XXXXX"
# Packet ID
packet_auth_token: "XXXXX"
# Google Mail ID
mail_secret: "secret_password"
from_secret: "mail@domain.com"
to_secret: "other_mail@domain.com"
# S3 files synchronization
S3_URL: s3.fr-par.scw.cloud
S3_REGION: FR-PAR
S3_ACCESS_KEY: "XXXXX"
S3_SECRET_KEY: "XXXXX"
```

It is strongly recommended to encrypt this file :

```bash
ansible-vault encrypt playbooks/vars/secret.yml
```

## Main parameters

Some main parameters are defined in the `playbooks/vars/main.yml` variables file.

```yaml
---
## General variables
# EasyRSA variables
easyrsa_generate_dh: true
easyrsa_servers:
  - name: server
easyrsa_clients:
  - name: client
easyrsa_pki_dir: /etc/easyrsa/pki
# OpenVPN variables
openvpn_use_pam: false
openvpn_client_to_client: false
openvpn_comp_lzo: false
openvpn_unified_client_profiles: true
openvpn_keydir: "{{ easyrsa_pki_dir }}"
openvpn_clients: "{{ easyrsa_clients }}"
openvpn_download_clients: true
openvpn_download_dir: /tmp/
openvpn_server: 172.16.253.0 255.255.255.0
openvpn_route_traffic: true
openvpn_route_ranges:
  - 192.168.122.0 255.255.255.0
# Docker variables
docker_users:
  - gns3
# GNS3-server variables
gns3s_host: "172.16.253.1"
gns3s_port: "3080"
gns3s_home: /home/gns3/
# Files synchronization
gns3s_files:
  - s3src: s3://labimages/gns3/images
    s3dst: "{{ gns3s_home }}"
  - s3src: s3://labimages/gns3/projects
    s3dst: "{{ gns3s_home }}"
#  - s3src: s3://labimages/gns3/appliances
#    s3dst: /usr/share/gns3/gns3-server/lib/python3.6/site-packages/gns3server
easyrsa_conf_req_country: FR
easyrsa_conf_req_cn: "{{ inventory_hostname }}"
easyrsa_conf_req_province: "Paris"
easyrsa_conf_req_city: "Paris"
easyrsa_conf_req_org: "GNS3 labs"
easyrsa_conf_req_email: "root@{{ inventory_hostname }}"
easyrsa_conf_req_ou: "gns3labs"

```

## Provision servers

_If you already have a server on the hand, you can avoid the `playbooks/provision.yml` playbook._

I use Cloudflare API with my domain to provide an easy name to remember for management. I you do not use a Cloudflare managed zone, please define the public IP address of your server in the `ansible_host` inventory variable.

GNS3 is in constant development but we need a robust installation with dev or stable latest releases. The latest Ubuntu Bionic can use the GNS3 Release Team PPA repos. No need to support any virtualbox, vpcs or IOU in my scenarios. KVM and Docker are sufficient.

You can choose several [Scaleway servers](https://blog.scaleway.com/2016/c2-insanely-affordable-x64-servers/) `scw_type` :

- "C2S"
- "C2M"
- "C2L"
- "DEV1-S" (only for my own convenience)

Note: [Scaleway offers also "premium" bare metal category](https://www.scaleway.com/fr/serveurs-bare-metal/). The provision function should be supported by a tool such as Terraform and integrated with Ansible configuration management. Terraform can be more "agile" for multi-vendors servers.

Or you can choose [Packet servers](https://www.packet.com/cloud/servers/) `packet_type` :

- "t1.small.x86" (default)
- "c1.small.x86"
- "t3.small.x86"
- "c3.small.x86"

Or you can choose any baremetal service provider, any real server or virtual machine (with nested virtualization) with a minimal Ubuntu Bionic (18.04) installation to use the following playbooks.

## Install GNS3 server

Roles embedded in the `playbooks/install-gns3-server.yml` playbook :

- "create a ca" (nkakouros.easyrsa)
- "install openvpn" (Stouts.openvpn)
- "install libvirtd" (install-libvirtd)
- "install docker-engine" (geerlingguy.docker)
- "install gns3-server" (install-gns3-server)
- "install fail2ban" (install-fail2ban)
- "enable routing" (enable-routing)

## Synchronize files

The `playbooks/synchronize-gns3-files.yml` synchronize several S3 bucket folder to the destination server. I use thoses variables to transfer my images and my projects to the servers (`playbooks/vars/main.yml`) :

```yaml
gns3s_files:
  - s3src: s3://labimages/gns3/images
    s3dst: "{{ gns3s_home }}"
  - s3src: s3://labimages/gns3/projects
    s3dst: "{{ gns3s_home }}"
```

Do not forget to fix and to protect the S3 credits  :

```yaml
# S3 files synchronization
S3_URL: s3.fr-par.scw.cloud
S3_REGION: FR-PAR
S3_ACCESS_KEY: "XXXXX"
S3_SECRET_KEY: "XXXXX"
```

## Send credits

The `playbooks/send-credits.yml` playbook send a custom message in french or in english with the OpenVPN connexion file and the process to install and configure an Openvpn client and the GNS3 client software.

Do not forget to fix and to protect the mail credits :

```yaml
# Google Mail ID
mail_secret: "secret_password"
from_secret: "mail@domain.com"
to_secret: "other_mail@domain.com"
```

## Ansible configuration

```ini
[defaults]
inventory = ./inventories/hosts
#inventory=./inventories/scaleway_inventory.yml
private_key_file = ~/.ssh/id_rsa
forks = 16
#strategy = free
gathering = explicit
become = True
host_key_checking = False
ssh_args = -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no
log_path = ./ansible.log
enable_plugins = host_list, script, yaml, ini, auto
vault_password_file = ~/.vault_passwords.txt
#display_ok_hosts = no
#display_skipped_hosts = no
callback_whitelist = profile_tasks
#[callback_profile_tasks]
#task_output_limit = 100

```
