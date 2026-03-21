## Ansible: MySQL (Master-Master Replication) - HAProxy - Zabbix Monitoring<br>

<div align="right">
  
**Язык:** [English](README.md) | [Русский](README.ru.md) 

</div> 


### Содержание:
* [Описание](#описание)
* [Стек](#стек)
* [Требования](#требования)
* [Структура репозитория](#структура-репозитория)
* [Как запустить сценарий](#как-запустить-сценарий)
* [Проверка](#проверка) 


### Описание:
Репозиторий содержит сценарий Ansible для автоматизированного развертывания высокодоступного кластера MySQL (Master-Master Replication) с балансировкой нагрузки (HAProxy) и системой мониторинга (Zabbix).

Сценарий включает:
  - Базовую подготовку узлов (часовой пояс, необходимые зависимости).
  - Установку и конфигурацию MySQL с Master-Master репликацией через GTID (Global Transaction Identifiers).
  - Настройку HAProxy для TCP-балансировки и обеспечения отказоустойчивости (failover) с активным мониторингом доступности узлов.
  - Развертывание Zabbix Server в Docker-контейнере с PostgreSQL и фиксированной сетью.
  - Установку Zabbix Agent 2 (APT) на всех узлах (MySQL, HAProxy, Zabbix Server).
  - Мониторинг MySQL через встроенный плагин Zabbix Agent 2.
  - Обеспечение безопасности с помощью UFW и Fail2Ban.


### Стек:
- Операционная система: Ubuntu 24.04 LTS
- База данных: MySQL Server 8+
- Балансировка, высокая доступность: HAProxy
- Мониторинг: Zabbix (Server, Web, PostgreSQL в Docker; Agent 2 на всех узлах)
- Контейнеризация: Docker, Docker Compose
- Безопасность: UFW, Fail2Ban, Ansible Vault


### Требования:
- Ubuntu 24.04 LTS[¹](#примечание-к-версиям)
- Ansible core 2.20.3[¹](#примечание-к-версиям)
- Python 3.12.3[¹](#примечание-к-версиям)
- Учетная запись с правами 'sudo', доступ по SSH-ключам.

### Примечание к версиям:
¹ Сценарий тестировался на указанных версиях. Использование более ранних или иных версий не гарантирует корректную работу и требует дополнительного тестирования.


### Структура репозитория:

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

### Как запустить сценарий:

1. Клонирование репозитория:
```
git clone git@github.com:Den4u/ha-mysql-zabbix.git
cd ha-mysql-zabbix
```

2. Настройка инвентаря: <br />
```
cp inventory/hosts.example inventory/hosts
```
Отредактируйте inventory/hosts

3. Проверка доступности серверов: <br />
```
ansible all -m ping
```

4. Создание зашифрованного хранилища секретов: <br />

```
ansible-vault create secrets.yml
```

Введите надежный пароль для Vault. Структура ожидаемых секретов приведена в secrets.yml.example

5. Настройка переменных окружения:  <br />

Все необходимые переменные распределены по файлам `group_vars/` и `host_vars/`.  <br />

Создайте файлы переменных, используя примеры с расширением .example:
```
cp group_vars/all_nodes.yml.example group_vars/all_nodes.yml
cp group_vars/mysql_nodes.yml.example group_vars/mysql_nodes.yml
cp host_vars/master1.yml.example host_vars/master1.yml
cp host_vars/master2.yml.example host_vars/master2.yml
cp host_vars/haproxy.yml.example host_vars/haproxy.yml
cp host_vars/zabbix_host.yml.example host_vars/zabbix_host.yml
```

Минимально необходимые переменные для корректного запуска (остальные параметры отредактируйте под вашу среду):
```
group_vars/all_nodes.yml:
 - ignoreip_f2b			      # IP для исключения из блокировки Fail2Ban
 - admin_ssh_ip			      # IP администратора для доступа по SSH
 - zabbix_host_ip	    	  # Приватный IP адрес Zabbix сервера
 - server_			          # Приватный  IP для пассивных проверок агента
 - server_active		      # Приватный  IP для активных проверок агента

group_vars/mysql_nodes.yml:
 - haproxy_host_ip		      # Приватный IP адрес HAProxy сервера
 
host_vars/haproxy.yml:
 - master1_ip			      # Приватный IP адрес первого MySQL master
 - master2_ip			      # Приватный IP адрес второго MySQL master

host_vars/master1.yml:
 - other_master_ip		      # Приватный IP второго MySQL master

host_vars/master2.yml:
 - other_master_ip		      # Приватный IP первого MySQL master
 
host_vars/zabbix_host.yml:
 - zabbix_server_host_bind	  # Приватный IP адрес Zabbix сервера
```
Более подробное описание переменных в соответствующих файлах.

<br />

6. Запуск плейбука: <br />

Запуск полного сценария:
```
ansible-playbook master.yml --ask-vault-password
```
Запуск отдельных компонентов:  <br />

```
--tags apt_cache	      # Обновление APT кэша на всех узлах

--tags common	          # Базовая настройка всех узлов (timezone, ufw, fail2ban)

--tags mysql	          # Установка и настройка MySQL Master-Master Replication

--tags haproxy	          # Установка и настройка HAProxy

--tags docker             # Установка Docker (Zabbix хост)

--tags zabbix-server      # Поднятие контейнеров (Zabbix server, Web, PostgreSQL)

--tags docker-zabbix	  # Общая (установка Docker + поднятие контейнеров)

--tags zabbix-agent	      # Установка Zabbix Agent 2 на всех узлах (APT)
```
Пример запуска:
```
ansible-playbook master.yml --ask-vault-password --tags mysql
```

### Проверка:

#### 1. Автоматическая проверка репликации:  <br />
После каждого запуска плейбука Ansible автоматически проверяет состояние репликации. Если параметры `Replica_IO_Running` или `Replica_SQL_Running` не равны 'Yes', плейбук прервет выполнение с детальным сообщением об ошибке.

Чтобы запустить только проверку статуса (без изменений):
```
ansible-playbook master.yml --ask-vault-pass --tags "check_replication"
```

#### 2. Ручная проверка репликации: <br />
Подключитесь по SSH к одному из серверов базы данных:
```
ssh user@master1
```
Проверьте статус MySQL:
```
systemctl status mysql
```
Проверьте статус репликации:
```
mysql -S /var/run/mysqld/mysqld.sock  <<EOF
SHOW MASTER STATUS;
SHOW REPLICA STATUS\G
SELECT @@GLOBAL.gtid_executed;
EOF
```

#### 3. Доступ к панелям управления:  <br />
Для безопасности сервисы настроены на прослушивание только локальных интерфейсов/приватной сети. Для доступа к веб-интерфейсам с вашей локальной машины рекомендуется использовать SSH-туннели.

#### Zabbix Web Interface:
```
ssh -L 8080:localhost:8080 user@zabbix_node_ip
```
После установки туннеля откройте в браузере: http://127.0.0.1:8080

Данные для входа по умолчанию:
  - Логин: Admin
  - Пароль: zabbix

#### HAProxy Stats (Admin Panel):
```
ssh -L 8081:localhost:8081 user@haproxy_node_ip
```
После установки туннеля откройте в браузере: http://127.0.0.1:8081/admin-panel

Данные для входа:
  - Логин: указан в secrets.yml
  - Пароль: указан в secrets.yml

<br />

### Реализация: [Den4u](https://github.com/Den4u)
