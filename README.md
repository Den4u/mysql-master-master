## Ansible: HAProxy + MySQL (Master-Master Replication)<br>
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Ansible](https://img.shields.io/badge/ansible-%231A1918.svg?style=for-the-badge&logo=ansible&logoColor=white)
![MySQL](https://img.shields.io/badge/mysql-4479A1.svg?style=for-the-badge&logo=mysql&logoColor=white)

### Содержание:
* [Описание](#описание)
* [Стек](#стек)
* [Требования](#требования)
* [Структура репозитория](#структура-репозитория)
* [Как запустить сценарий](#как-запустить-сценарий)
* [Проверка](#проверка) 


### Описание:
Репозиторий содержит сценарий Ansible для развертывания HAProxy + MySQL (Master-Master). 

Настройка включает:
  - Базовую подготовку системы (часовой пояс, необходимые зависимости).
  - Установку и конфигурацию MySQL с Master-Master репликацией через GTID.
  - Настройку HAProxy для TCP-балансировки и обеспечения отказоустойчивости (failover) с активным мониторингом доступности узлов.
  - Обеспечение безопасности с помощью UFW и Fail2Ban.


### Стек:
- Операционная система: Ubuntu 24.04 LTS
- База данных: MySQL Server 8+
- Балансировка нагрузки и отказоустойчивость: HAProxy
- Безопасность: UFW, Fail2Ban
- Репликация 	GTID (Global Transaction Identifiers)


### Требования:
- Ubuntu 24.04 LTS[¹](#примечание-к-версиям)
- Ansible core 2.20.3[¹](#примечание-к-версиям)
- Python 3.12.3[¹](#примечание-к-версиям)
- OpenSSH_9.6p1[¹](#примечание-к-версиям): учетная запись с правами 'sudo', доступ по SSH-ключам.

### Примечание к версиям:
¹ Сценарий тестировался на указанных версиях. Использование более ранних или иных версий не гарантирует корректную работу и требует дополнительного тестирования.


### Структура репозитория:

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

### Как запустить сценарий:

1. Клонирование репозитория:
```
git clone git@github.com:Den4u/mysql-master-master.git
cd mysql-master-master
```

2. Настройка инвентаря: <br />

Создайте файл инвентаря ./inventory/hosts (или используйте hosts.example). Укажите в нем IP-адреса серверов, данные пользователей и SSH-порты. <br />


3. Проверка доступности серверов: <br />
```
ansible all -m ping
```

4. Создание зашифрованного хранилища секретов: <br />

Создайте файл secrets.yml в корне репозитория с использованием Ansible Vault.
```
ansible-vault create secrets.yml
```

Введите надежный пароль для Vault. Структура ожидаемых секретов приведена в ./secrets.yml.example

5. Настройка переменных окружения:  <br />

В каталоге group_vars создайте файл all.yml, а в host_vars — файлы для каждого хоста: master1.yml, master2.yml, haproxy_b.yml. <br />
В качестве примеров используйте файлы с расширением .example в соответствующих каталогах.
Отредактируйте переменные под вашу среду.

<br />

6. Задайте необходимые параметры для fail2ban: <br />
```
- logpath_ssh:    # Путь к логу SSH.
- maxretry_f2b:   # Максимальное количество попыток входа перед блокировкой.
- findtime_f2b:   # Период времени, в течение которого засчитываются неудачные попытки.
- bantime_f2b:    # Продолжительность блокировки (например, 1d, 1h, 10m).
- ignoreip_f2b:   # IP-адреса, которые не будут блокироваться (укажите через пробел).
- port_ssh_f2b:   # Порт SSH.
```

7. Запуск плейбука: <br />

```
ansible-playbook master.yml --ask-vault-password
```

### Проверка:

 <br />

1. Автоматическая проверка репликации:  <br />
После каждого запуска плейбука Ansible автоматически проверяет состояние репликации. Если параметры Replica_IO_Running или Replica_SQL_Running не равны 'Yes', плейбук прервет выполнение с детальным сообщением об ошибке.

Чтобы запустить только проверку статуса (без изменений):
```
ansible-playbook master.yml --ask-vault-pass -v --tags "check_replication"
```

2. Ручная проверка репликации: <br />
Подключитесь по SSH к одному из мастеров:
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
SHOW SLAVE STATUS\G
SELECT @@GLOBAL.gtid_executed;
EOF
```

3. Мониторинг HAProxy:  <br />
Панель HAProxy для отслеживания состояния бэкендов доступна по HTTP:
```
http://<IP_HAPROXY>:8080/{{ admin_panel }}
```
Безопасность: Доступ разрешён только с IP-адресов, указанных в переменной allowed_network. <br />
Авторизация: Используются логин и пароль, заданные в secrets.yml

### Реализация: [Den4u](https://github.com/Den4u)
