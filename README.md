## Ansible: MySQL (Master-Master Replication)<br>
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Ansible](https://img.shields.io/badge/ansible-%231A1918.svg?style=for-the-badge&logo=ansible&logoColor=white)
![MySQL](https://img.shields.io/badge/mysql-4479A1.svg?style=for-the-badge&logo=mysql&logoColor=white)

### Содержание:
* [Описание](#описание)
* [Стек](#стек)
* [Требования](#требования)
* [Структура репозитория](#структура-репозитория)
* [Как запустить сценарий](#как-запустить-сценарий)
* [Проверка репликации](#проверка-репликации) 


### Описание:
Репозиторий содержит готовый сценарий `master.yml` для автоматической настройки двух серверов MySQL в режиме Active-Active Master-Master (M-M) репликации.

Настройка включает:
  - Установку часового пояса и необходимых зависимостей.
  - Установку и базовую конфигурацию MySQL.
  - Настройку репликации с использованием Global Transaction Identifiers (GTID).
  - Применение базовых мер безопасности: настройка брандмауэра UFW (с открытием портов только для доверенных узлов) и установка Fail2Ban.


### Стек:
- Операционная система: Ubuntu 24.04 LTS[¹](#примечание-к-версиям)
- База данных: MySQL Server Ver 8.0.45[¹](#примечание-к-версиям)
- Безопасность: UFW 0.36.2[¹](#примечание-к-версиям), Fail2Ban v1.0.2[¹](#примечание-к-версиям)
- Репликация 	GTID (Global Transaction Identifiers)


### Требования:
- Ubuntu 24.04 LTS[¹](#примечание-к-версиям)
- Ansible core 2.16.3[¹](#примечание-к-версиям)
- Python 3.12.3[¹](#примечание-к-версиям)
- OpenSSH_9.6p1[¹](#примечание-к-версиям): Учетная запись, от имени которой запускается Ansible, должна иметь права `sudo` на целевых хостах. Настоятельно рекомендуется использовать SSH-ключи.

### Примечание к версиям:
¹ Указанные версии являются протестированными. Использование более ранних версий (например, Ubuntu 20.04/22.04 или Ansible < 2.16) не гарантирует успешное выполнение сценария.


### Структура репозитория:

```
.
├── ansible.cfg
├── group_vars
│   └── all.yml.example
├── host_vars
│   ├── master1.yml.example
│   └── master2.yml.example
├── inventory
│   └── hosts.example
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
└──  secrets.yml.example

```

### Как запустить сценарий:

1. Клонирование репозитория:
```
git clone git@github.com:Den4u/mysql-master-master.git
cd mysql-master-master
```

2. Настройка инвентаря: <br />

Создайте файл инвентаря ./inventory/hosts (или используйте hosts.example). В этом файле укажите IP-адреса удаленных серверов, данные пользователей, SSH-порты. <br />


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

5. Настройка переменных окружения (host_vars / group_vars):  <br />

В каталогах group_vars и host_vars создайте файлы переменных ./group_vars/all.yml , ./host_vars/{master1.yml,master2.yml}

Также вы можете использовать примеры:  all.yml.example , master1.yml.example , master2.yml.example <br />

Измените значения переменных на актуальные для вашей среды.  <br />

Задайте необходимые параметры для fail2ban:
```
- logpath_ssh:    # Путь к логу SSH.
- maxretry_f2b:   # Максимальное количество попыток входа перед блокировкой.
- findtime_f2b:   # Период времени, в течение которого засчитываются неудачные попытки.
- bantime_f2b:    # Продолжительность блокировки (например, 1d, 1h, 10m).
- ignoreip_f2b:   # IP-адреса, которые не будут блокироваться (укажите через пробел).
- port_ssh_f2b:   # Порт SSH.
```

7. Запуск плейбука: <br />
Запустите плейбук. Сценарий автоматически проверит статус репликации после запуска:
```
ansible-playbook master.yml --ask-vault-password
```

### Проверка репликации:

 <br />
- Автоматическая проверка статуса: включена. Если Replica_IO_Running или Replica_SQL_Running не равно 'Yes' после запуска, выполнение прервется, сигнализируя о сбое репликации. <br />

Запуск только проверки статуса:
```
ansible-playbook master.yml --ask-vault-pass -v --tags "check_replication"
```
<br />
- Ручная проверка:  <br />
```
ssh user@master1
```
Проверка статуса сервиса: <br />
```
systemctl status mysql
```

Проверка статуса репликации:

```
mysql -S /var/run/mysqld/mysqld.sock  <<EOF
SHOW MASTER STATUS;
SHOW SLAVE STATUS\G
SELECT @@GLOBAL.gtid_executed;
EOF
```

### Реализация: [Den4u](https://github.com/Den4u)
