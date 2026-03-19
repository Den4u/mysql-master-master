## Ansible: HAProxy + MySQL (Master-Master Replication)<br>

<div align="right">
  
**Language:** [English](README.md) | [Русский](README.ru.md) 

</div> 


### Contents:
* [Description](#description)
* [Tech Stack](#tech-stack)
* [Requirements](#requirements)
* [Repository Structure](#repository-structure)
* [How to Run the Playbook](#how-to-run-the-playbook)
* [Verification](#verification) 


### Description:
This repository contains an Ansible playbook for deploying HAProxy + MySQL (Master-Master). 

The configuration includes:
  - Basic system preparation (timezone, required dependencies).
  - MySQL installation and configuration with Master-Master replication using GTID.
  - HAProxy setup for TCP load balancing and failover with active node health monitoring.
  - Security hardening using UFW and Fail2Ban.


### Tech Stack:
- Operating System: Ubuntu 24.04 LTS
- Database: MySQL Server 8+
- Load Balancing and Failover: HAProxy
- Security: UFW, Fail2Ban
- Replication: GTID (Global Transaction Identifiers)


### Requirements:
- Ubuntu 24.04 LTS[¹](#version-notes)
- Ansible core 2.20.3[¹](#version-notes)
- Python 3.12.3[¹](#version-notes)
- OpenSSH_9.6p1[¹](#version-notes): User account with 'sudo' privileges, SSH key-based access.

### Version Notes:
¹ The playbook has been tested with the specified versions. Using earlier or different versions may not guarantee proper functionality and requires additional testing.


### Repository Structure:

```
.
├── ansible.cfg
├── group_vars
│   └── all.yml.example
├── host_vars
│   ├── haproxy_b.yml.example
│   ├── master1.yml.example
│   └── master2.yml.example
├── inventory
│   └── hosts.example
├── LICENSE
├── master.yml
├── README.md
├── README.ru.md
├── roles
│   ├── dependencies
│   │   └── tasks
│   │       └── main.yml
│   ├── fail2ban
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── jail.local.j2
│   ├── haproxy
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── haproxy.cfg.j2
│   ├── mysql
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── mysqld.cnf.j2
│   ├── timezone
│   │   └── tasks
│   │       └── main.yml
│   └── ufw
│       └── tasks
│           └── main.yml
└── secrets.yml.example

```

### How to Run the Playbook:

1. Clone the repository:
```
git clone git@github.com:Den4u/mysql-master-master.git
cd mysql-master-master
```

2. Configure the inventory: <br />

Create the inventory file ./inventory/hosts (or use hosts.example). Specify server IP addresses, user credentials, and SSH ports. <br />


3. Verify server connectivity: <br />
```
ansible all -m ping
```

4. Create an encrypted secrets vault: <br />

Create a secrets.yml file in the repository root using Ansible Vault.
```
ansible-vault create secrets.yml
```

Enter a strong Vault password. The expected secrets structure is provided in ./secrets.yml.example

5. Configure environment variables:  <br />

In the group_vars directory, create an all.yml file, and in host_vars, create files for each host: master1.yml, master2.yml, haproxy_b.yml. <br />
Use the .example files in the respective directories as templates.
Edit the variables according to your environment.

<br />

6. Set required fail2ban parameters: <br />
```
- logpath_ssh:    # Path to the SSH log file.
- maxretry_f2b:   # Maximum number of failed login attempts before blocking.
- findtime_f2b:   # Time period during which failed attempts are counted.
- bantime_f2b:    # Block duration (e.g., 1d, 1h, 10m).
- ignoreip_f2b:   # IP addresses that will not be blocked (space-separated).
- port_ssh_f2b:   # SSH port.
```

7. Run the playbook: <br />

```
ansible-playbook master.yml --ask-vault-password
```

### Verification:

 <br />

1. Automatic replication check:  <br />
After each playbook run, Ansible automatically verifies the replication status. If Replica_IO_Running or Replica_SQL_Running are not 'Yes', the playbook will abort with a detailed error message.

To run only the status check (without changes):
```
ansible-playbook master.yml --ask-vault-pass -v --tags "check_replication"
```

2. Manual replication check: <br />
SSH into one of the masters:
```
ssh user@master1
```
Check MySQL status:
```
systemctl status mysql
```
Check replication status:
```
mysql -S /var/run/mysqld/mysqld.sock  <<EOF
SHOW MASTER STATUS;
SHOW SLAVE STATUS\G
SELECT @@GLOBAL.gtid_executed;
EOF
```

3. HAProxy monitoring:  <br />
The HAProxy dashboard for backend status monitoring is available via HTTP:
```
http://<IP_HAPROXY>:8080/{{ admin_panel }}
```
Security: Access is restricted to IP addresses specified in the allowed_network variable. <br />
Authentication: Uses the login and password defined in secrets.yml

### Implementation by: [Den4u](https://github.com/Den4u)
