# Настройка лабораторного стенда

Состояние на 28 сентября 2026 года: установлен и проверен контроллер `dc1`. Виртуальные машины `pc1` и `pc2` созданы, но ОС на них ещё не установлены.

Документ описывает настройку нового стенда. На уже работающем `dc1` повторять установку и `samba-tool domain provision` не требуется. История выполненных действий ведётся в [журнале](journal.md), результаты и ограничения разобраны в [исследовании](research.md).

Порядок настройки DNS ниже уточнён после обнаруженной ошибки: при повторении инструкции системный DNS переключается на Samba до первого запуска службы. В выполненной работе переключение было сделано после первого запуска. Полная повторная установка по этой редакции инструкции пока не проводилась.

## 1. Оборудование и состав стенда

Хост: Windows 11 Home x64, AMD Ryzen 7 250, 16 ГБ RAM, SSD 1000 ГБ. На начало работы свободно 767 ГБ. Гипервизор — VirtualBox 7.2.16 r174877.

ВМ  | ОС                                            | RAM     | vCPU | Виртуальный диск | Назначение
----|-----------------------------------------------|---------|------|------------------|----------------------------------------
dc1 | ALT Server 11.1, редакция «Альт Домен»        | 4096 МБ | 2    | 40 ГБ            | Контроллер домена
pc1 | ALT Workstation 11.2, установка запланирована | 3072 МБ | 2    | 40 ГБ            | Клиент и рабочее место администратора с ADMC
pc2 | ALT Workstation 11.2, установка запланирована | 3072 МБ | 2    | 40 ГБ            | Второй клиент для сравнительных проверок

Для дисков выбран VDI с динамическим выделением места. Для `dc1` графическое окружение не устанавливалось. Локальный репозиторий находится в `C:\alt-domain-lab`, ISO — в `C:\OS-lab\iso`.

## 2. Установочные образы

Образы получены с официальных страниц [ALT Server](https://www.basealt.ru/alt-server/download) и [ALT Workstation](https://www.basealt.ru/alt-workstation/download).

В PowerShell на Windows:

```powershell
Get-FileHash "C:\OS-lab\iso\alt-server-11.1-x86_64.iso" -Algorithm SHA512 | Format-List Path,Hash
Get-FileHash "C:\OS-lab\iso\alt-workstation-11.2-x86_64.iso" -Algorithm SHA512 | Format-List Path,Hash
```

Полученные суммы совпали с опубликованными для этих образов. Перенос строки в выводе PowerShell не является частью хеша.

`alt-server-11.1-x86_64.iso`:

```text
24367d9090efc42d1987b9cb155338e1ec00f86c5e2e31e5cde226c1f1b791836a8228749bedcc903b935b25dac5ccf6f1b7463279332ee096c26cb3fe754e0d
```

`alt-workstation-11.2-x86_64.iso`:

```text
b919de7f108a88d7f84ca710739820d3e9a465d8269b58aae86091557d90b861832c2466961949eb71fedc5c408c854157d4c3dd058f7b7ec4ba23d8a33ae5a7
```

## 3. Сеть VirtualBox и адреса

В менеджере сетей VirtualBox создана **Сеть NAT** с именем `ad-lab` и подсетью `10.77.0.0/24`. У каждой ВМ адаптер 1 подключён к этой сети; виртуальный кабель подключён. Общая сеть NAT обеспечивает связь между ВМ и выход во внешнюю сеть.

Узел | Полное имя   | IP            | DNS        | Состояние
-----|--------------|---------------|------------|----------
dc1  | dc1.lab.test | 10.77.0.10/24 | 127.0.0.1  | Настроен
pc1  | pc1.lab.test | 10.77.0.21/24 | 10.77.0.10 | План
pc2  | pc2.lab.test | 10.77.0.22/24 | 10.77.0.10 | План

Шлюз в этом стенде — `10.77.0.1`. На этапе установки `dc1` использовался DHCP. Перед переходом на статические адреса DHCP сети `ad-lab` отключается; для клиентов также планируется ручная настройка.

До развёртывания домена использовался DNS `172.19.41.204`, полученный в данном окружении. Это адрес конкретного стенда, а не универсальный DNS для других компьютеров. При воспроизведении следует определить доступный внешний DNS командой `cat /etc/resolv.conf` до отключения DHCP и проверить разрешение внешних имён. Далее этот адрес используется как `dns forwarder` Samba. Не следует заменять его адресом `127.0.0.1` в параметре `dns forwarder`.

## 4. Установка dc1

1. Подключить серверный ISO к оптическому приводу `dc1` и выполнить ручную установку.
2. В загрузочном меню выбрать `Install ALT Server 11.1 x86_64`.
3. Выбрать русский язык и редакцию «Альт Домен».
4. Проверить дату, время и часовой пояс `Europe/Moscow`.
5. Выбрать диск `sda — VBOX HARDDISK`, 40 ГБ, и профиль установки сервера.
6. Проверить предложенную разметку: `sda1` около 3918 МБ под swap; `sda2` около 36 ГБ, Ext4, точка монтирования `/`.
7. Из дополнительных групп оставить серверную поддержку виртуальных окружений. GNOME, legacy web-интерфейс и группу клиентской инфраструктуры Samba AD не выбирать.
8. Установить загрузчик на весь диск `sda`.
9. Задать имя компьютера `dc1.lab.test`; на этом этапе оставить DHCP.
10. Задать пароль `root` и создать локальную учётную запись. В стенде используется `gudzichek-dc1`.
11. После завершения установки отключить ISO от виртуального оптического привода и загрузиться с виртуального диска.

После установки ожидается текстовый вход в систему. Локальная учётная запись и будущий доменный `Administrator` — разные пользователи.

В выполненной установке ISO остался подключённым, поэтому после перезагрузки снова открылся установщик. Извлечение ISO решило проблему; переустановка не потребовалась.

После первого входа проверены `hostname`, `ip -br addr`, `ip route`, доступность шлюза и разрешение внешних имён. Создан снимок `01-clean-install`.

## 5. Постоянный IP через etcnet

Дальнейшие команды настройки выполняются в консоли `dc1`, а не в PowerShell или Git Bash. Для получения прав администратора:

```bash
su -
```

Перед изменениями проверено состояние:

```bash
ip -br addr
ip route
cat /etc/net/ifaces/enp0s3/options
cat /etc/resolv.conf
systemctl is-active network NetworkManager systemd-networkd
```

В стенде интерфейс называется `enp0s3`. Служба `network` активна, остальные две неактивны. В `options` заданы `NM_CONTROLLED=no` и `SYSTEMD_CONTROLLED=no`: настройками управляет etcnet. Для другой конфигурации эти команды изменения не следует применять без проверки.

После отключения DHCP в настройках сети VirtualBox:

```bash
cp -a /etc/net/ifaces/enp0s3 /root/enp0s3-before-static
sed -i 's/^BOOTPROTO=.*/BOOTPROTO=static/' /etc/net/ifaces/enp0s3/options
echo '10.77.0.10/24' > /etc/net/ifaces/enp0s3/ipv4address
echo 'default via 10.77.0.1' > /etc/net/ifaces/enp0s3/ipv4route
echo 'nameserver 172.19.41.204' > /etc/net/ifaces/enp0s3/resolv.conf
systemctl restart network
```

Внешний DNS в последней команде записи должен соответствовать окружению. Знак `>` перезаписывает файл. Проверка:

```bash
ip -br addr
ip route
ping -c 3 10.77.0.1
getent ahostsv4 download.basealt.ru
```

Ожидаются `10.77.0.10/24`, маршрут через `10.77.0.1`, ответы шлюза и IP-адреса сервера загрузок. Способ настройки основан на [документации etcnet](https://docs.altlinux.org/ru-RU/alt-server/11.1/html/alt-server/etcnet-static.html).

## 6. Обновление системы и проверка времени

Выполнять команды последовательно, дожидаясь завершения каждой:

```bash
apt-get update
apt-get dist-upgrade
update-kernel
apt-get clean
reboot
```

При ошибке обновления сначала разобраться с ней, а не продолжать весь список. После перезагрузки:

```bash
uname -r
ip -br addr
ip route
getent ahostsv4 download.basealt.ru
timedatectl
```

Зафиксированное ядро — `6.12.110-6.12-alt1`. Адрес сохранился, DNS работал. `timedatectl` показал `System clock synchronized: yes`, `NTP service: active`, часовой пояс `Europe/Moscow`. Выдача времени клиентам с `dc1` пока не настроена и не проверена.

При повторении позже репозитории могут содержать другие версии. Действия обновления соответствуют [руководству ALT](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/install-distro--update-after-install--chapter.html).

## 7. Установка Samba и подготовка конфигурации

Под `root`:

```bash
apt-get install task-samba-dc
rpm -q task-samba-dc samba-dc
```

В выполненной установке оба пакета имеют версию `4.21.9-alt4.p11.1`. Выбран вариант Samba DC с Heimdal Kerberos. Основание — [раздел установки пакетов](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/deploy-samba-tool.html).

Перед созданием домена:

```bash
testparm -s
ls -l /var/lib/samba/private/sam.ldb
systemctl is-active samba smb nmb winbind krb5kdc slapd bind
```

На новом стенде `testparm` показал `ROLE_STANDALONE`, `sam.ldb` отсутствовала, все перечисленные службы были неактивны. Если база уже существует, это другой исходный сценарий: повторное создание домена по этой инструкции не выполнять. Если работают отдельные конфликтующие службы, подготовить их отключение по [руководству](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/ostanovka_konfliktujuschih_sluzhb.html); на наблюдаемом стенде необходимости останавливать работающие службы не было.

Сохранить стандартный конфигурационный файл:

```bash
mv /etc/samba/smb.conf /etc/samba/smb.conf.before-domain
```

## 8. Создание домена

```bash
samba-tool domain provision
```

Вопрос                   | Ответ для стенда
-------------------------|------------------------------------------------
Realm                    | LAB.TEST
Domain                   | LAB
Server Role              | dc
DNS backend              | SAMBA_INTERNAL
DNS forwarder IP address | 172.19.41.204 — доступный DNS данного окружения
Administrator password   | Собственный пароль, вводится интерактивно
Retype password          | Повтор пароля

Пароль не включается в команды, скриншоты и Git. Учётная запись `Administrator` принадлежит домену. В выполненном создании дополнительные параметры уровня домена и `--use-rfc2307` явно не задавались; фактический функциональный уровень отдельно ещё не проверялся.

Признаки успешного создания: `Server Role: active directory domain controller`, `Hostname: dc1`, `NetBIOS Domain: LAB`, `DNS Domain: lab.test` и выведенный SID домена. SID в другой установке будет отличаться.

## 9. Kerberos и DNS самого контроллера

Сохранить прежнюю конфигурацию Kerberos и установить файл, сформированный Samba:

```bash
cp -a /etc/krb5.conf /root/krb5.conf.before-domain
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

Это копирование файла, а не создание символической ссылки. Способ описан в [настройке Kerberos](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/nastrojka_kerberos.html).

Переключить системные DNS-запросы на встроенный DNS Samba:

```bash
cp -a /etc/net/ifaces/enp0s3/resolv.conf /root/resolv.conf.before-domain
echo 'nameserver 127.0.0.1' > /etc/net/ifaces/enp0s3/resolv.conf
echo 'search lab.test' >> /etc/net/ifaces/enp0s3/resolv.conf
systemctl restart network
systemctl enable --now samba
```

Между переключением DNS и запуском Samba разрешение имён может временно не работать. При успешном запуске проверить:

```bash
systemctl is-active samba
cat /etc/resolv.conf
samba-tool domain info 127.0.0.1
```

Ожидается `active`. В `/etc/resolv.conf` должны быть `nameserver 127.0.0.1` и `search lab.test`. Файл `/etc/resolv.conf` формируется автоматически; постоянная настройка в этом стенде хранится в каталоге интерфейса etcnet.

Внешний DNS остаётся в параметре `dns forwarder` Samba; в системный список DNS после создания домена он больше не добавляется. Основание — [сетевые настройки контроллера](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/setevye_nastrojki.html).

## 10. Приёмочные проверки dc1

### DNS

```bash
host dc1.lab.test
host -t SRV _ldap._tcp.lab.test
host -t SRV _kerberos._udp.lab.test
getent ahostsv4 download.basealt.ru
samba_dnsupdate
echo $?
```

`echo $?` нужно выполнять сразу после `samba_dnsupdate`. Ожидаемые результаты:

Проверка         | Результат
-----------------|-----------------------------------------------
Имя dc1.lab.test | 10.77.0.10
SRV LDAP         | Порт 389, dc1.lab.test
SRV Kerberos     | Порт 88, dc1.lab.test
Внешнее имя      | Разрешается; при проверке получен 176.12.98.77
samba_dnsupdate  | Код 0

Адрес внешнего сайта может меняться. Существенно успешное разрешение, а не совпадение с сохранённым адресом.

### Kerberos и ресурсы SMB

```bash
kinit Administrator@LAB.TEST
klist
smbclient -L dc1.lab.test -U Administrator
```

Ввести пароль доменного `Administrator` по запросу. `klist` должен показать принципал `Administrator@LAB.TEST` и билет `krbtgt/LAB.TEST@LAB.TEST`. В списке SMB-ресурсов ожидаются `sysvol`, `netlogon` и `IPC$`. Строка `SMB1 disabled -- no workgroup available` не отменяет успешный вывод ресурсов.

Получение TGT и просмотр ресурсов здесь являются отдельными проверками. Команда `smbclient` с паролем сама по себе не доказывает, что SMB-сеанс использовал Kerberos.

Проверки опираются на [руководство ALT](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/testing-samba-dc.html). Фактические результаты приведены в [исследовании](research.md) и [подборке подтверждений](../evidence/2026-09-28/README.md).

## 11. Проверка перезагрузки и снимок

```bash
reboot
```

После входа и `su -`:

```bash
systemctl is-active samba
ip -br addr
cat /etc/resolv.conf
samba-tool domain info 127.0.0.1
getent ahostsv4 download.basealt.ru
samba_dnsupdate
echo $?
```

В выполненной проверке служба запустилась, IP и DNS сохранились, домен отвечал, внешнее имя разрешалось, `samba_dnsupdate` вернула 0.

Для сохранения состояния:

```bash
systemctl poweroff
```

После выключения ВМ в VirtualBox открыть «Снимки» и создать `02-domain-ready`. На 28 сентября сохранены оба снимка: `01-clean-install` и `02-domain-ready`. Второй сделан до подключения клиентов. Он фиксирует состояние ВМ, но не заменяет отдельную резервную копию файлов стенда.

## 12. Что ещё предстоит

- Настроить и проверить синхронизацию времени клиентов с выбранным источником; при использовании dc1 подготовить его как NTP-сервер.
- Установить pc1 и pc2, назначить запланированные адреса и DNS 10.77.0.10.
- Ввести клиентов в домен, проверить вход одного доменного пользователя на обеих машинах.
- Установить ADMC и GPUI на pc1, записать их версии и способ подключения клиентов.
- Провести эксперименты с учётными записями, группами и политиками.

Инструкции для этих шагов будут добавляться после выполнения и проверки. Текущие успешные тесты проведены на самом dc1; связь клиентов с контроллером ещё не проверена.

## Источники

Дата обращения: 28 сентября 2026 года. Ссылки на конкретные разделы приведены рядом с соответствующими действиями.

- [Руководство Альт Домен 11.1](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/index.html).
- [Создание домена со встроенным DNS Samba](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/ch10s06s04.html).
- [Сети VirtualBox 7.2](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html).
- [Снимки VirtualBox 7.2](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/working-with-vms.html).
