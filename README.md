# Снятие статистики fail2ban с помощью zabbix 6.0

## Настройка zabbix-client
Создаем файл для `fail2ban`:

```bash
$ sudo nano /etc/zabbix/zabbix_agentd.conf.d/fail2ban.conf
```

И записываем туда следующее содержимое:
```conf
UserParameter=fail2ban.discovery,sudo fail2ban-client status | grep 'Jail list:' | sed -e 's/^.*:\W\+//' -e 's/\(\w\+\)/{"{#JAIL}":"\1"}/g' -e 's/.*/{"data":[\0]}/'
UserParameter=fail2ban.total_failed[*],sudo fail2ban-client status $1 | grep 'Total failed:' | grep -E -o '[0-9]+'
UserParameter=fail2ban.total_banned[*],sudo fail2ban-client status $1 | grep 'Total banned:' | grep -E -o '[0-9]+'
UserParameter=fail2ban.currently_failed[*],sudo fail2ban-client status $1 | grep 'Currently failed:' | grep -E -o '[0-9]+'
UserParameter=fail2ban.currently_banned[*],sudo fail2ban-client status $1 | grep 'Currently banned:' | grep -E -o '[0-9]+'
```

Перезапускаем zabbix-agent:
```bash
$ sudo service zabbix-agent restart
```

Проверим, применились ли настройки
```bash
$ sudo zabbix_agentd -p | grep ^fail2ban

fail2ban.discovery                            [t|{"data":[{"{#JAIL}":"sshd"}]}]
fail2ban.total_failed                         [t|]
fail2ban.total_banned                         [t|]
fail2ban.currently_failed                     [t|]
fail2ban.currently_banned                     [t|]
```

Попробуем получить значения
```bash
$ sudo zabbix_agentd -t 'fail2ban.total_failed[sshd]'
fail2ban.total_failed[sshd]                   [t|2556]
$ sudo zabbix_agentd -t 'fail2ban.total_banned[sshd]'
fail2ban.total_banned[sshd]                   [t|922]
$ sudo zabbix_agentd -t 'fail2ban.currently_failed[sshd]'
fail2ban.currently_failed[sshd]               [t|11]
$  sudo zabbix_agentd -t 'fail2ban.currently_banned[sshd]'
fail2ban.currently_banned[sshd]               [t|466]
```

---

### Доступ zabbix к fail2ban

Так как `fail2ban` работает от пользователя `root`, то нужно разрешить доступ для `zabbix` к `fail2ban`.

Для этого, в файл `/etc/sudoers` добавляем следующее содержимое:

```conf
# zabbix
zabbix ALL=NOPASSWD: /usr/bin/fail2ban-client status
zabbix ALL=NOPASSWD: /usr/bin/fail2ban-client status *
```

Альтернативный вариант - запускать `zabbix` от пользователя `root` (менее предпочтительный). Для это в файл `/etc/zabbix/zabbix_agentd.conf` нужно добавить 2 параметра:

```conf
AllowRoot=1
User=root
```
---

## zabbix-server

Осталось только импортировать файл `template_fail2ban.yaml` в шаблоны zabbix-server и указать для `Узла сети` наш шаблон - `Templates` > `Fail2ban`.

---
Источники:
 - https://github.com/zabbix/community-templates/tree/main/Applications/Firewall/template_fail2ban/6.0
 - https://github.com/hermanekt/zabbix-fail2ban-discovery-
