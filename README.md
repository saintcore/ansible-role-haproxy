saintcore.haproxy
=========

[![lint](https://github.com/saintcore/ansible-role-haproxy/actions/workflows/lint.yml/badge.svg)](https://github.com/saintcore/ansible-role-haproxy/actions/workflows/lint.yml)
[![molecule](https://github.com/saintcore/ansible-role-haproxy/actions/workflows/molecule.yml/badge.svg)](https://github.com/saintcore/ansible-role-haproxy/actions/workflows/molecule.yml)

This role installs and configures **HAProxy** (High Availability Proxy) as a robust load balancer and reverse proxy. Optionally, it sets up:

* **Keepalived**: For high-availability clustering, managing virtual router redundancy protocol (VRRP) to ensure the service's VIP remains active even if a node fails.
* **Coraza-SPOA**: A high-performance Web Application Firewall (WAF) using the HAProxy Stream Processing Offload Agent (SPOA) protocol. It comes pre-integrated with the **Core Rule Set (CRS)** to provide protection against common web attack categories (like SQL Injection, XSS, etc.).

Requirements
------------

This role has the following prerequisites:

* **Ansible:** Requires Ansible **v2.10** or higher.
* **Privilege Escalation:** The role's tasks use **become: true internally**. The user running the playbook must be able to escalate privileges (e.g., be a user configured for sudo access) on the target node. **You do not need to set become: true in your playbook.**
* **Dependencies:** The role relies on the following Ansible Galaxy collections, which must be installed on your control node:
    * community.general
    * ansible.posix

Role Variables
--------------

## haproxy

| Variable                      | Description                   | Default                       | Defined in        |
| :---                          | :---                          | :---                          | :---              |
| haproxy_maps                  | (opt) maps file to copy       | []                            | defaults/main.yml |
| haproxy_rsyslog               | (req) rsyslog config          | 'etc_rsyslog.haproxy.conf.j2' | defaults/main.yml |
| haproxy_config                | (req) main config file        | 'etc_haproxy_haproxy.cfg.j2'  | defaults/main.yml |
| haproxy_nbthread              | (opt) number of threads       | 1                             | defaults/main.yml |
| haproxy_seport_http_custom    | (opt) selinux-ports to allow  | []                            | defaults/main.yml |
| haproxy_package_name          | package name to install       | haproxy                       | see vars folder   |
| haproxy_pin_version           | packae version to insall      | "2.4"                         | see vars folder   |
| haproxy_apparmor_config   | apparmor config for debian    | 'etc_apparmor.d_local_usr.sbin.haproxy.j2' | see vars folder |

## keepalived

keepalived / cluster related tasks only run if **haproxy_cluster** is **true**. Note: If clustering is enabled, the variables below are all required.

| Variable                      | Description                   | Default                               | Defined in        |
| :---                          | :---                          | :---                                  | :---              |
| haproxy_cluster               | install and configure cluster | false                                 | defaults/main.yml |
| haproxy_keepalived_config     | main config file              | 'etc_keepalived_keepalived.conf.j2'   | defaults/main.yml |
| haproxy_keepalived_state      | state (master/backup) per node |                                      | defaults/main.yml |
| haproxy_keepalived_priority   | priority per node             |                                       | defaults/main.yml |
| haproxy_keepalived_pass       | pass for auth betwen nodes    |                                       | defaults/main.yml |
| haproxy_keepalived_virtualip  | virtualip to use for active node |                                    | defaults/main.yml |

## coraza-spoa

coraza-spoa / WAF related tasks only run if **haproxy_coraza** is **true**. Note: Most variables are required when enabling Coraza-SPOA, as you really should provide adopted configuration files for your specific use cases.

| Variable                      | Description                   | Default                               | Defined in        |
| :---                          | :---                          | :---                                  | :---              |
| haproxy_coraza                | install and configure coraza  | false                                 | defaults/main.yml |
| haproxy_coraza_version        | git release/branch to install | 'main'                                | defaults/main.yml |
| haproxy_coraza_crs_version    | git release/branch to install | 'main'                                | defaults/main.yml |
| haproxy_coraza_port           | port to use                   | 'main'                                | defaults/main.yml |
| haproxy_coraza_sitesconfigdirs | dirs per sit to creae for crs | []                                   | defaults/main.yml |
| haproxy_coraza_sitesconfigfiles | config files per site to copy for crs | []                          | defaults/main.yml |
| haproxy_coraza_configyaml   | main config file for coraza   | 'coraza/etc_coraza-spoa_config.yaml.j2' | defaults/main.yml |
| haproxy_coraza_conf         | 'modsecurity' setup           | 'coraza/etc_coraza-spoa_coraza.conf.j2' | defaults/main.yml |
| haproxy_coraza_spoaconf       | haproxy-config for coraza     | 'coraza/etc_haproxy_coraza.cfg.j2'    | defaults/main.yml |
| haproxy_coraza_owasp_csrconf  | main crs setup                | 'coraza/etc_coraza_crs-setup.conf.j2' | defaults/main.yml |
| haproxy_coraza_systemd      | systemd config     | 'coraza/etc_systemd_system_coraza-spoa.service.j2' | defaults/main.yml |

## coraza-spoa-selinux

On SELinux-enabled systems, you can optionally install and use a policy module. These tasks only run if **haproxy_coraza_selinux** is **true**. Note: If enabled, the version var is also required. See the dedicated repository for more information: [selinux-coraza-spoa | github.com](https://github.com/saintcore/selinux-coraza-spoa)

| Variable                      | Description                   | Default                               | Defined in        |
| :---                          | :---                          | :---                                  | :---              |
| haproxy_coraza_selinux        | install and use policy module | false                                 | defaults/main.yml |
| haproxy_coraza_selinux_v      | git release/branch to install |                                       | defaults/main.yml |

Dependencies
------------

This role requires the following Ansible Galaxy collections:

- community.general
- ansible.posix


Example Playbooks
----------------

## haproxy and keepalived

### vars
host_vars/host1.yml

```yaml
haproxy_keepalived_state: 'MASTER'
haproxy_keepalived_priority: 110
```

host_vars/host2.yml

```yaml
haproxy_keepalived_state: 'BACKUP'
haproxy_keepalived_priority: 100

```

host_vars/host3.yml

```yaml
haproxy_keepalived_state: 'BACKUP'
haproxy_keepalived_priority: 90
```

### Playbook

```yaml
- hosts: host1, host2, host3
  roles:
    - role: saintcore.haproxy
      haproxy_config: 'haproxy.cfg.j2'
      haproxy_nbthread: 4
      haproxy_seport_http_custom: [1200, 1201, 1203] # all ports used for your haproxy-backends
      haproxy_cluster: true
      haproxy_keepalived_pass: 'yoursecurepass'
      haproxy_keepalived_virtualip: '10.10.160.101' # ip used for "main" instance (vip)
```

## haproxy and coraza-spoa

This example uses coraza-spoa as a waf.


```yaml
- hosts: host1, host2, host3
  roles:
    - role: saintcore.haproxy
      haproxy_config: 'haproxy.cfg.j2'
      haproxy_nbthread: 4
      haproxy_seport_http_custom: [1200, 1201, 1203] # all ports used for your haproxy-backends
      haproxy_coraza: true
      haproxy_coraza_port: 9000
      haproxy_coraza_selinux: true
      haproxy_coraza_selinux_v: 'v1.1.1'
```

License
-------

* This role is licensed under **AGPL-3.0-only**.
* This identifier is defined by the SPDX project. You can view the specific license details here: [AGPL-3.0-only on SPDX](https://spdx.org/licenses/AGPL-3.0-only.html)

Author Information
------------------

This role was created by **saintcore**. You can find more of my work on GitHub: [github.com/saintcore](https://github.com/saintcore)