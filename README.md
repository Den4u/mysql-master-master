## Ansible: MySQL (Master-Master Replication) - HAProxy - Zabbix Monitoring<br>

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
This repository contains an Ansible playbook for automated deployment of a highly available MySQL cluster (Master-Master Replication) with load balancing (HAProxy) and monitoring system (Zabbix).

The playbook includes:
  - Base system setup (timezone, required dependencies)
  - MySQL installation and configuration with Master-Master replication using GTID (Global Transaction Identifiers)
  - HAProxy configuration for TCP load balancing and failover with active node health monitoring
  - Zabbix Server deployment in Docker containers with PostgreSQL and a fixed network
  - Zabbix Agent 2 installation (APT) on all nodes (MySQL, HAProxy, Zabbix Server)
  - MySQL monitoring via the built-in Zabbix Agent 2 plugin
  - Security hardening with UFW and Fail2Ban


### Tech Stack:
- Operating System: Ubuntu 24.04 LTS
- Database: MySQL Server 8+
- Load Balancing, High Availability: HAProxy
- Monitoring: Zabbix (Server, Web, PostgreSQL in Docker; Agent 2 on all nodes)
- Containerization: Docker, Docker Compose
- Security: UFW, Fail2Ban, Ansible Vault


### Requirements:
- Ubuntu 24.04 LTS[¹](#version-notes)
- Ansible core 2.20.3[¹](#version-notes)
- Python 3.12.3[¹](#version-notes)
- A user account with 'sudo' privileges and SSH key-based access

### Version Notes:
¹ The playbook has been tested with the specified versions. Using earlier or different versions may not guarantee proper functionality and requires additional testing.


### Repository Structure:

```
.
├── ansible.cfg
├── group_vars
│   ├── all_nodes.yml.example
│   └── mysql_nodes.yml.example
├── host_vars
│   ├── haproxy.yml.example
│   ├── master1.yml.example
│   ├── master2.yml.example
│   └── zabbix_host.yml.example
├── inventory
│   └── hosts.example
├── LICENSE
├── master.yml
├── README.md
├── README.ru.md
├── roles
│   ├── common
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   ├── fail2ban.yml
│   │   │   ├── main.yml
│   │   │   ├── timezone.yml
│   │   │   └── ufw.yml
│   │   └── templates
│   │       └── jail.local.j2
│   ├── docker
│   │   └── tasks
│   │       └── main.yml
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
│   ├── zabbix-agent
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── mysql.conf.j2
│   │       └── zabbix_agent2.conf.j2
│   └── zabbix-server
│       ├── handlers
│       │   └── main.yml
│       ├── tasks
│       │   └── main.yml
│       └── templates
│           └── docker-compose.yml.j2
└── secrets.yml.example

```

### How to Run the Playbook:

1. Clone the repository:
```
git clone git@github.com:Den4u/ha-mysql-zabbix.git
cd ha-mysql-zabbix
```

2. Configure the inventory: <br />
```
cp inventory/hosts.example inventory/hosts
```
Edit `inventory/hosts`

<br />

3. Verify server connectivity: <br />
```
ansible all -m ping
```

4. Create an encrypted secrets vault: <br />

Create a secrets.yml file in the repository root using Ansible Vault.
```
ansible-vault create secrets.yml
```

Enter a strong Vault password. The expected secrets structure is provided in `secrets.yml.example`

5. Configure environment variables:  <br />

All required variables are distributed across `group_vars/` and `host_vars/` files. <br />

Create variable files using the `.example` templates:
```
cp group_vars/all_nodes.yml.example group_vars/all_nodes.yml
cp group_vars/mysql_nodes.yml.example group_vars/mysql_nodes.yml
cp host_vars/master1.yml.example host_vars/master1.yml
cp host_vars/master2.yml.example host_vars/master2.yml
cp host_vars/haproxy.yml.example host_vars/haproxy.yml
cp host_vars/zabbix_host.yml.example host_vars/zabbix_host.yml
```
Minimally required variables for successful deployment (edit other parameters according to your environment):

```
group_vars/all_nodes.yml:
 - ignoreip_f2b			      # IP to exclude from Fail2Ban blocking
 - admin_ssh_ip			      # Admin workstation IP for SSH access
 - zabbix_host_ip	    	  # Private IP address of Zabbix server
 - server_			          # Private IP for passive agent checks
 - server_active		      # Private IP for active agent checks

group_vars/mysql_nodes.yml:
 - haproxy_host_ip		      # Private IP address of HAProxy server
 
host_vars/haproxy.yml:
 - master1_ip			      # Private IP address of first MySQL master
 - master2_ip			      # Private IP address of second MySQL master

host_vars/master1.yml:
 - other_master_ip		      # Private IP of second MySQL master

host_vars/master2.yml:
 - other_master_ip		      # Private IP of first MySQL master
 
host_vars/zabbix_host.yml:
 - zabbix_server_host_bind	  # Private IP address of Zabbix server
```

See the corresponding files for detailed variable descriptions.
<br />

6. Run the playbook: <br />

Run the complete deployment:

```
ansible-playbook master.yml --ask-vault-password
```

Run individual components using tags:  <br />
```
--tags apt_cache	      # Update APT cache on all nodes

--tags common	          # Base configuration on all nodes (timezone, ufw, fail2ban)

--tags mysql	          # Install and configure MySQL Master-Master Replication

--tags haproxy	          # Install and configure HAProxy

--tags docker             # Install Docker (Zabbix host)

--tags zabbix-server      # Deploy containers (Zabbix server, Web, PostgreSQL)

--tags docker-zabbix	  # Combined (install Docker + deploy containers)

--tags zabbix-agent	      # Install Zabbix Agent 2 on all nodes (APT)
```
<br />

Example:
```
ansible-playbook master.yml --ask-vault-password --tags mysql
```

### Verification:

#### 1. Automatic replication check:  <br />
After each playbook run, Ansible automatically verifies the replication status. If `Replica_IO_Running` or `Replica_SQL_Running` are not 'Yes', the playbook will abort with a detailed error message.

To run only the status check (without making changes):
```
ansible-playbook master.yml --ask-vault-pass -v --tags "check_replication"
```

#### 2. Manual replication check: <br />
Connect via SSH to one of the database servers:
```
ssh user@master1
```
Check MySQL service status:
```
systemctl status mysql
```
Check replication status:
```
mysql -S /var/run/mysqld/mysqld.sock  <<EOF
SHOW MASTER STATUS;
SHOW REPLICA STATUS\G
SELECT @@GLOBAL.gtid_executed;
EOF
```

#### 3. Access to management panels:  <br />
For security, services are configured to listen only on local interfaces or private networks. To access the web interfaces from your local machine, it is recommended to use SSH tunnels.

#### Zabbix Web Interface:
```
ssh -L 8080:localhost:8080 user@zabbix_node_ip
```
After establishing the tunnel, open your browser at: http://127.0.0.1:8080

Default login credentials:
  - Username: Admin
  - Password: zabbix

#### HAProxy Stats (Admin Panel):
```
ssh -L 8081:localhost:8081 user@haproxy_node_ip
```
After establishing the tunnel, open your browser at: http://127.0.0.1:8081/admin-panel

Login credentials:
  - Username: specified in `secrets.yml`
  - Password: specified in `secrets.yml`

<br />

### Implementation by: [Den4u](https://github.com/Den4u)
