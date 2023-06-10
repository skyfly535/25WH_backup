# Резервное копирование.

Задание:

- создать директорию для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;

- репозиторий для резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;

- имя бекапа должно содержать информацию о времени снятия бекапа;

- глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;

- резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;

- настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.


Цель:

Научиться настраивать удаленный бекап каталога c оного сервера на другой при помощи borgbackup.

## Развертывание стенда для демострации настройки FreeIPA.

Стэнд состоит из хостовой машины под управлением ОС Ubuntu 20.04 на которой развернут Ansible и двух виртуальных машин `centos/7`.

Машина с именем: `backup` выполняет роль сервера сбора и хранения резервных копий.

Машина с именем: `client` выполняет роль клиента (с которой собирают логи).

Разворачиваем инфраструктуру в Vagrant исключительно через Ansible.

Все коментарии по каждому блоку указаны в тексте Playbook - `back.yml`.

Файлы `borg-backup.service.j2`, `borg-backup.timer.j2` и каталог `roles` необходимо поместить в каталог с Playbook `back.yml` и Vagranfile.

Выполняем установку стенда:

```
vagrant up
```
## Проверяем работу стенда

Подключимся к ВМ `client`: 
```
vagrant ssh client
```
и перейдём в root-пользователя: sudo -i.

Смотрим список резервных копий (имя копии состоит из каталога резервирования, даты  и времени создания копии)

```
[root@client vagrant]# borg list borg@backup:/var/backup/
Enter passphrase for key ssh://borg@backup/var/backup: 
etc-2023-06-10_19:09:29              Sat, 2023-06-10 19:09:30 [f35785ec35b46b2193d5236aa933d2e185a4c59dba19087a04fa6e52a57e9551]
```
Для дальнейшей проверки останавливаем службу таймера резервного копирования:

```
[root@client vagrant]# systemctl stop borg-backup.timer  
[root@client vagrant]# systemctl status borg-backup.timer
● borg-backup.timer - Borg Backup
   Loaded: loaded (/etc/systemd/system/borg-backup.timer; enabled; vendor preset: disabled)
   Active: inactive (dead) since Sat 2023-06-10 19:38:55 +10; 7s ago

Jun 10 18:35:26 client systemd[1]: Started Borg Backup.
Jun 10 19:38:55 client systemd[1]: Stopped Borg Backup.
```
Удаляем каталог /etc:

```
[root@client ~]# rm -rf /etc
rm: cannot remove '/etc': Device or resource busy
```
Смотрим, что каталог действительно пуст:

```
[root@client ~]# ls -al /etc
total 0
drwxr-xr-x.  2 0 0   6 Jun 10 10:08 .
dr-xr-xr-x. 18 0 0 255 Jun 10 08:34 ..
```

Необходимо добавить конфигурацию пользователя `root` (так как она была в каталоге `etc`) для соединения с удаленным сервером:

```
[root@client ~]# echo "root:x:0:0:root:/root:/bin/bash" > /etc/passwd
```

так же нам необходимо восстановить файл `etc/hosts`:

```
[root@client ~]# echo "192.168.11.160 backup" > /etc/hosts
```

Выполняем запрос к репозиторию

```
[root@client ~]# borg list borg@backup:/var/backup/
Enter passphrase for key ssh://borg@backup/var/backup: 
etc-2023-06-10_19:36:49              Sat, 2023-06-10 09:36:50 [7d088b41390d9fe7013022407a385f8e2d3fa7aa3a9c24996aa130dbfbc001e9]
```
Производим восстановление каталога `etc` из резервной копии

```
[root@client /]# cd /; borg extract borg@backup:/var/backup/::etc-2023-06-10_19:36:49
Enter passphrase for key ssh://borg@backup/var/backup: 
Warning: File system encoding is "ascii", extracting non-ascii filenames will not be supported.
Hint: You likely need to fix your locale setup. E.g. install locales and use: LANG=en_US.UTF-8
[root@client /]# ls -al /etc
total 1072
drwxr-xr-x. 78 root root     8192 Jun 10 19:10 .
dr-xr-xr-x. 18 root root      255 Jun 10 18:34 ..
-rw-------.  1 root root        0 May  1  2020 .pwd.lock
-rw-r--r--.  1 root root      163 May  1  2020 .updated
-rw-r--r--.  1 root root     5090 Aug  6  2019 DIR_COLORS
-rw-r--r--.  1 root root     5725 Aug  6  2019 DIR_COLORS.256color
-rw-r--r--.  1 root root     4669 Aug  6  2019 DIR_COLORS.lightbgcolor
-rw-r--r--.  1 root root       94 Mar 25  2017 GREP_COLORS
drwxr-xr-x.  7 root root      134 Apr  2  2020 NetworkManager
drwxr-xr-x.  5 root root       57 May  1  2020 X11
-rw-r--r--.  1 root root       16 May  1  2020 adjtime
-rw-r--r--.  1 root root     1529 Apr  1  2020 aliases
-rw-r--r--.  1 root root    12288 Jun 10 18:33 aliases.db
drwxr-xr-x.  2 root root     4096 May  1  2020 alternatives
-rw-------.  1 root root      541 Aug  9  2019 anacrontab
drwxr-x---.  3 root root       43 May  1  2020 audisp
drwxr-x---.  3 root root       83 Jun 10 18:33 audit
drwxr-xr-x.  2 root root       68 May  1  2020 bash_completion.d
-rw-r--r--.  1 root root     2853 Apr  1  2020 bashrc
drwxr-xr-x.  2 root root        6 Apr  8  2020 binfmt.d
-rw-r--r--.  1 root root       37 Apr  8  2020 centos-release
-rw-r--r--.  1 root root       51 Apr  8  2020 centos-release-upstream
drwxr-xr-x.  2 root root        6 Aug  4  2017 chkconfig.d
-rw-r--r--.  1 root root     1108 Aug  8  2019 chrony.conf
...
```
Проверим логирование процесса бекапа:

```
[root@client ~]# journalctl -u borg-backup.service
-- Logs begin at Sat 2023-06-10 20:59:04 +10, end at Sat 2023-06-10 21:13:03 +10. --
Jun 10 21:00:35 client systemd[1]: Starting Borg Backup...
Jun 10 21:00:38 client borg[4317]: ------------------------------------------------------------------------------
Jun 10 21:00:38 client borg[4317]: Archive name: etc-2023-06-10_21:00:35
Jun 10 21:00:38 client borg[4317]: Archive fingerprint: cf49cff267df17e0b30989b80b00e410dcf47abd71a72ae8717240367ba534a6
Jun 10 21:00:38 client borg[4317]: Time (start): Sat, 2023-06-10 21:00:36
Jun 10 21:00:38 client borg[4317]: Time (end):   Sat, 2023-06-10 21:00:38
Jun 10 21:00:38 client borg[4317]: Duration: 1.95 seconds
Jun 10 21:00:38 client borg[4317]: Number of files: 1700
Jun 10 21:00:38 client borg[4317]: Utilization of max. archive size: 0%
Jun 10 21:00:38 client borg[4317]: ------------------------------------------------------------------------------
Jun 10 21:00:38 client borg[4317]: Original size      Compressed size    Deduplicated size
Jun 10 21:00:38 client borg[4317]: This archive:               28.43 MB             13.49 MB             11.84 MB
Jun 10 21:00:38 client borg[4317]: All archives:               28.43 MB             13.49 MB             11.84 MB
Jun 10 21:00:38 client borg[4317]: Unique chunks         Total chunks
Jun 10 21:00:38 client borg[4317]: Chunk index:                    1283                 1699
Jun 10 21:00:38 client borg[4317]: ------------------------------------------------------------------------------
Jun 10 21:00:41 client systemd[1]: Started Borg Backup.
Jun 10 21:06:16 client systemd[1]: Starting Borg Backup...
Jun 10 21:06:18 client borg[6419]: ------------------------------------------------------------------------------
Jun 10 21:06:18 client borg[6419]: Archive name: etc-2023-06-10_21:06:17
Jun 10 21:06:18 client borg[6419]: Archive fingerprint: f8b9de4d5b7bcf91dbdfdf8293e482ceacb561bd2385b7256d35ba01f5cbbf75
Jun 10 21:06:18 client borg[6419]: Time (start): Sat, 2023-06-10 21:06:17
Jun 10 21:06:18 client borg[6419]: Time (end):   Sat, 2023-06-10 21:06:18
Jun 10 21:06:18 client borg[6419]: Duration: 0.29 seconds
Jun 10 21:06:18 client borg[6419]: Number of files: 1701
Jun 10 21:06:18 client borg[6419]: Utilization of max. archive size: 0%
Jun 10 21:06:18 client borg[6419]: ------------------------------------------------------------------------------
Jun 10 21:06:18 client borg[6419]: Original size      Compressed size    Deduplicated size
Jun 10 21:06:18 client borg[6419]: This archive:               28.43 MB             13.49 MB            127.18 kB
Jun 10 21:06:18 client borg[6419]: All archives:               56.85 MB             26.98 MB             11.97 MB
Jun 10 21:06:18 client borg[6419]: Unique chunks         Total chunks
Jun 10 21:06:18 client borg[6419]: Chunk index:                    1292                 3399
Jun 10 21:06:18 client borg[6419]: ------------------------------------------------------------------------------
Jun 10 21:06:20 client systemd[1]: Started Borg Backup.
Jun 10 21:12:16 client systemd[1]: Starting Borg Backup...
Jun 10 21:12:18 client borg[8131]: ------------------------------------------------------------------------------
Jun 10 21:12:18 client borg[8131]: Archive name: etc-2023-06-10_21:12:17
Jun 10 21:12:18 client borg[8131]: Archive fingerprint: 6b5428a66df9928200fe36e88d892be708d78ca11d06f57b6e75c2dab2ce7533
Jun 10 21:12:18 client borg[8131]: Time (start): Sat, 2023-06-10 21:12:17
Jun 10 21:12:18 client borg[8131]: Time (end):   Sat, 2023-06-10 21:12:18
Jun 10 21:12:18 client borg[8131]: Duration: 0.28 seconds
Jun 10 21:12:18 client borg[8131]: Number of files: 1701
Jun 10 21:12:18 client borg[8131]: Utilization of max. archive size: 0%
Jun 10 21:12:18 client borg[8131]: ------------------------------------------------------------------------------
Jun 10 21:12:18 client borg[8131]: Original size      Compressed size    Deduplicated size
Jun 10 21:12:18 client borg[8131]: This archive:               28.43 MB             13.49 MB             26.39 kB
Jun 10 21:12:18 client borg[8131]: All archives:               56.86 MB             26.99 MB             11.87 MB
Jun 10 21:12:18 client borg[8131]: Unique chunks         Total chunks
Jun 10 21:12:18 client borg[8131]: Chunk index:                    1286                 3400
Jun 10 21:12:18 client borg[8131]: ------------------------------------------------------------------------------
Jun 10 21:12:20 client systemd[1]: Started Borg Backup.
```