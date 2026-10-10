# Настройка лабораторного стенда

Состояние на 10 октября 2026 года: контроллер и два клиента работают; выполнены E01/E02, E09 на pc2 и часть E05 с обоями на обоих клиентах. Инструкция дополнена настройкой GPO и исправлением NTP. Съёмные носители ещё не проверены.

Документ описывает настройку нового стенда. На уже работающем `dc1` повторять установку и `samba-tool domain provision` не требуется. История выполненных действий ведётся в [журнале](journal.md), результаты и ограничения разобраны в [исследовании](research.md).

Порядок настройки DNS ниже уточнён после обнаруженной ошибки: при повторении инструкции системный DNS переключается на Samba до первого запуска службы. В выполненной работе переключение было сделано после первого запуска. Полная повторная установка по этой редакции инструкции пока не проводилась.

## 1. Оборудование и состав стенда

Хост: Windows 11 Home x64, AMD Ryzen 7 250, 16 ГБ RAM, SSD 1000 ГБ. На начало работы свободно 767 ГБ. Гипервизор — VirtualBox 7.2.16 r174877.

ВМ  | ОС                                     | RAM     | vCPU | Виртуальный диск | Назначение
----|----------------------------------------|---------|------|------------------|---------------------------------------------
dc1 | ALT Server 11.1, редакция «Альт Домен» | 4096 МБ | 2    | 40 ГБ            | Контроллер домена
pc1 | ALT Workstation 11.2                   | 3072 МБ | 2    | 40 ГБ            | Клиент и рабочее место администратора с ADMC
pc2 | ALT Workstation 11.2                   | 3072 МБ | 2    | 40 ГБ            | Второй клиент для сравнительных проверок

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
-----|--------------|---------------|------------|---------------
dc1  | dc1.lab.test | 10.77.0.10/24 | 127.0.0.1  | Настроен
pc1  | pc1.lab.test | 10.77.0.21/24 | 10.77.0.10 | Настроен; SSSD
pc2  | pc2.lab.test | 10.77.0.22/24 | 10.77.0.10 | Настроен; SSSD

Шлюз в этом стенде — `10.77.0.1`. На этапе установки `dc1` использовался DHCP. Перед переходом на статические адреса DHCP сети `ad-lab` отключается; клиенты используют ручную настройку.

В первоначальной установке использовался внешний DNS `172.19.41.204`. После проблемы с разрешением внешних имён 30 сентября на dc1 установлен `dns forwarder = 1.1.1.1`. Прямые запросы к `1.1.1.1` были проверены с dc1 и pc1.

3 октября в другой внешней сети forwarder заменён на проверенный `172.17.2.11`; клиентский DNS остался `10.77.0.10`. В командах первоначального развёртывания ниже сохранён адрес-пример `1.1.1.1`. Перед повторением проверить его доступность командой `host ftp.altlinux.org 1.1.1.1`; при необходимости выбрать доступный внешний DNS своего окружения. После создания домена системный DNS dc1 — `127.0.0.1`, клиентский — `10.77.0.10`. Внешний DNS указывается в `dns forwarder`, чтобы доменные запросы продолжали обрабатываться контроллером.

### Изменение forwarder на уже созданном dc1

Выполненные 30 сентября команды под root:

```bash
cp -a /etc/samba/smb.conf /root/smb.conf.before-dns-forwarder-change
sed -i 's/^[[:space:]]*dns forwarder[[:space:]]*=.*/    dns forwarder = 1.1.1.1/' /etc/samba/smb.conf
testparm -s --parameter-name='dns forwarder'
testparm -s && systemctl restart samba
```

Перед перезапуском проверка параметра должна вернуть выбранный внешний DNS. Последняя команда перезапускает Samba только после успешной проверки конфигурации. При выполнении была заменена уже существующая строка `dns forwarder`; повторное создание домена не выполнялось.

Подтверждения: [изменение на dc1](../evidence/30-09-2026/04-dc1-dns-forwarder-change.png), [успешное разрешение на pc1](../evidence/30-09-2026/05-pc1-dns-restored.png). Механизм описан в [разделе SAMBA_INTERNAL](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/ch10s06s04.html).

### Повторная смена forwarder 3 октября

В текущей внешней сети на dc1 сработал прямой запрос:

```bash
host -W 2 ftp.altlinux.org 172.17.2.11
```

После сохранения копии smb.conf существующая строка dns forwarder изменена на 172.17.2.11, конфигурация проверена и Samba перезапущена. На pc1 выполнены:

```bash
host -W 2 ftp.altlinux.org 10.77.0.10
apt-get update
```

Обе проверки завершились успешно. [Исходные подтверждения](../evidence/03-10-2026/README.md). При повторении в другой сети сначала проверить доступность выбранного внешнего DNS; последний проверенный адрес этого этапа не является универсальной настройкой.

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
echo 'nameserver 1.1.1.1' > /etc/net/ifaces/enp0s3/resolv.conf
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
-------------------------|--------------------------------------------------
Realm                    | LAB.TEST
Domain                   | LAB
Server Role              | dc
DNS backend              | SAMBA_INTERNAL
DNS forwarder IP address | 1.1.1.1 — проверить доступность в своём окружении
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

Проверки опираются на [руководство ALT](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/testing-samba-dc.html). Фактические результаты приведены в [исследовании](research.md) и [подборке подтверждений](../evidence/28-09-2026/README.md).

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

## 12. Установка клиентов pc1 и pc2

Используется тот же ISO ALT Workstation 11.2. ВМ устанавливаются отдельно. dc1 должен быть доступен во время сетевых проверок и подключения к домену.

Параметр               | pc1           | pc2
-----------------------|---------------|--------------
Имя компьютера         | pc1.lab.test  | pc2.lab.test
Статический IP         | 10.77.0.21/24 | 10.77.0.22/24
Шлюз                   | 10.77.0.1     | 10.77.0.1
DNS                    | 10.77.0.10    | 10.77.0.10
Домен поиска           | lab.test      | lab.test
Локальный пользователь | gudzichek-pc1 | gudzichek-pc2

При ручной установке выбрать компоненты «Клиенты сервисов → Альт Домен» и компоненты групповых политик: инструменты администрирования, клиент и шаблоны. Задать пароль root и локального пользователя соответствующей машины. После завершения извлечь ISO и загрузиться с установленного виртуального диска.

На pc2 1 октября выбор «Альт Домен» был пропущен. Нужные компоненты установлены после первого входа. В терминале клиента:

```bash
su -
apt-get update
apt-get install task-auth-ad-sssd alterator-auth gpupdate alterator-gpupdate
rpm -q task-auth-ad-sssd alterator-auth gpupdate alterator-gpupdate
```

`su -` запрашивает локальный пароль root этой ВМ. Пакет `task-auth-ad-sssd` нужен для подключения через SSSD; `alterator-auth` — модуль аутентификации ЦУС; `gpupdate` и `alterator-gpupdate` обеспечивают применение и включение групповых политик. Основание: [SSSD](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/client-sssd.html) и [компоненты групповых политик](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/alterator.html).

В запросе пакетов на pc2 до общего обновления получены: `task-auth-ad-sssd-0.52-alt1`, `alterator-auth-0.52-alt1`, `gpupdate-0.16.1-alt1`, `alterator-gpupdate-1.3-alt1`. Это версии на момент снимка; после последующих обновлений их следует запрашивать заново.

## 13. Сеть и обновление клиентов

Проверить на соответствующем клиенте:

```bash
hostname -f
ip -br addr
ip route
cat /etc/resolv.conf
host -t SRV _ldap._tcp.lab.test
host ftp.altlinux.org
```

Ожидаются имя и IP из таблицы, шлюз `10.77.0.1`, DNS `10.77.0.10`, поиск `lab.test`, SRV-запись с `dc1.lab.test` на порту 389 и успешное разрешение внешнего имени. Эти команды служат порядком проверки при повторении; полный вывод всех команд для pc2 в текущую подборку не включён.

При ошибке внешнего DNS проверить работу forwarder на dc1 по разделу 3. Системный DNS клиента оставляется доменным: `10.77.0.10`.

После успешной сетевой проверки под root выполнить последовательно:

```bash
apt-get update
apt-get dist-upgrade
update-kernel
apt-get clean
reboot
```

После перезагрузки под локальным пользователем:

```bash
uname -r
timedatectl
```

При проверках 30 сентября и 1 октября на клиентах зафиксированы ядро `6.12.110-6.12-alt1`, синхронизация времени `yes`, служба NTP `active` и часовой пояс `Europe/Moscow`. Тогда источник отдельно не проверялся. Его диагностика и исправление 10 октября описаны в разделе 23.

## 14. Ввод клиентов в домен через SSSD

В графическом сеансе локального пользователя открыть **Центр управления системой → Пользователи → Аутентификация**. При запросе пароля администратора ОС используется локальный пароль root клиента.

Выбрать **Active Directory или Альт Домен**:

Поле               | pc1                          | pc2
-------------------|------------------------------|------------------------------
Домен              | lab.test                     | lab.test
Рабочая группа     | LAB                          | LAB
Имя компьютера     | pc1                          | pc2
Способ подключения | SSSD (в единственном домене) | SSSD (в единственном домене)

Нажать «Применить». В диалоге подключения указать пользователя `Administrator` и пароль администратора домена, заданный при создании домена на dc1. При наличии флажка «Включить групповые политики» включить его. После сообщения об успешном подключении перезагрузить клиента.

Основание: [ввод через ЦУС](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/alterator-auth.html). В стенде оба клиента подключены через SSSD. Конкретная GPO проверена 10 октября в части обоев; см. раздел 24.

## 15. Тестовый пользователь и проверка входа

Для первого создания пользователя на dc1 под root:

```bash
samba-tool user create student1
```

Пароль задаётся интерактивно. В нашем домене пользователь уже создан 30 сентября; для входа на pc2 использовалась эта же учётная запись.

Перед входом на клиенте можно проверить получение данных:

```bash
getent passwd student1@lab.test
id student1@lab.test
```

Завершить локальный графический сеанс. На экране входа нажать «Нет в списке?» и ввести `student1` с его доменным паролем. Короткое имя успешно использовалось в выполненных входах; подробнее — [руководство ALT](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/dm-login.html).

В терминале нового пользовательского сеанса, без `su -`:

```bash
hostname -f
whoami
id
echo "$HOME"
klist
systemctl is-active sssd
```

Ожидаются правильное имя клиента, `student1`, его UID/GID и группы, домашний каталог, TGT `krbtgt/LAB.TEST@LAB.TEST` для `student1@LAB.TEST`, а также SSSD `active`.

В текущем домене UID student1 — `1164001104`, основной GID — `1164000513`, каталог — `/home/LAB.TEST/student1`. При новой установке домена числовые идентификаторы могут отличаться. Критерий сравнения — одна доменная учётная запись и согласованная идентификация на двух клиентах.

Одинаковый путь каталога на двух ВМ не создаёт общего хранилища. В выполненной проверке E01 вход на pc1 проверен 30 сентября, на pc2 — 1 октября. Исходные результаты: [pc1](../evidence/30-09-2026/pc1-domain-login.txt) и [pc2](../evidence/01-10-2026/pc2-domain-login.txt).

## 16. Проверка объектов на контроллере

На dc1 под root:

```bash
samba-tool computer list
systemctl is-active samba
```

В выполненной проверке получены `PC1$`, `DC1$`, `PC2$` и `active`. Порядок вывода списка не является критерием успеха. Команда списка описана в [руководстве Samba 4.21](https://www.samba.org/samba/docs/4.21/man-html/samba-tool.8.html).

Подтверждение: [список компьютеров](../evidence/01-10-2026/05-dc1-computers.png). Этот запрос дополняет проверку реального входа, а не заменяет её.

## 17. Предупреждение ядра и сохранение состояния

1 октября на dc1 обнаружено предупреждение `soft lockup` на 28 секунд. Для получения контекста выполнено:

```bash
journalctl -k -b --no-pager -n 40
systemctl is-active samba
```

В видимом фрагменте есть один эпизод в 16:48:54; Samba после него активна. Причина не установлена. Для дальнейшего наблюдения сохранён [исходный скриншот](../evidence/01-10-2026/06-dc1-kernel-warning.png). Интерпретация ограничена этим фрагментом; механизм описан в [документации ядра](https://docs.kernel.org/admin-guide/lockup-watchdogs.html).

Перед опытами с ADMC и политиками следует сохранить рабочее состояние машин. Клиенты выключаются штатно через меню ОС; на dc1 под root:

```bash
systemctl poweroff
```

После выключения в VirtualBox открыть «Снимки → Сделать». Снимки `01-domain-login-ok` на pc2 и `04-two-clients-ready` на dc1 позже показаны в присланных 9 октября изображениях менеджеров снимков. Более поздние точки после E02/E09 описаны в разделе 21; ранние снимки dc1 от 28 сентября — в разделе 11.

Для начала работы с ADMC достаточно запустить dc1 и pc1; pc2 можно включить при сравнительных опытах. Это уменьшает число одновременно работающих ВМ на хосте с 16 ГБ RAM.


## 18. Подключение ADMC на pc1

На pc1 ADMC уже был установлен. Проверка:

```bash
rpm -q admc
```

Получено admc-0.24.0-alt1.x86_64. В терминале локального пользователя:

```bash
kinit Administrator@LAB.TEST
klist
admc
```

Пароль доменного Administrator вводится интерактивно. klist должен показать его принципал и действующий TGT. В выполненном запуске ADMC отображает dc1.lab.test и administrator@lab.test.

Сначала проверить существующие объекты: student1 в Users, PC1/PC2 в Computers, DC1 в Domain Controllers. [Результаты подключения](../evidence/03-10-2026/README.md).

## 19. Создание пользователя, группы и членства — E02

dc1 должен быть доступен. На pc1 ADMC открыт под доменным Administrator.

1. В дереве домена создать подразделение Lab.
2. В Lab открыть «Создать → Пользователь».
3. Указать имя и полное имя student2, имя для входа student2 с суффиксом lab.test и короткое имя student2 после `LAB\`.
4. Задать тестовый пароль и подтверждение. Для первоначального входа снять «Пользователь должен сменить пароль при следующем входе». Остальные флажки в выполненном опыте оставлены снятыми.
5. Подтвердить создание и проверить student2 в Lab.
6. В Lab открыть «Создать → Группа». Имя — lab-students; область — «Глобальная»; тип — «Безопасность».
7. Открыть свойства созданной группы → «Участники» → «Добавить…», выбрать student2.
8. Нажать «Применить» и «ОК». Повторно открыть свойства и проверить участника.

Видимый участник в ещё не сохранённом окне не заменяет серверной проверки: в выполненном опыте первые команды на dc1 показывали пустое членство, а после сохранения повторные команды подтвердили изменение.

На dc1 под root:

```bash
samba-tool user show student2
samba-tool group show lab-students
samba-tool group listmembers lab-students
```

В выполненном результате DN пользователя — CN=student2,OU=Lab,DC=lab,DC=test. Его memberOf указывает на lab-students. Группа содержит member с DN student2, а listmembers выводит student2.

Это шаги для новых тестовых объектов. В текущем домене объекты уже созданы; повторное создание для продолжения работы не требуется.

## 20. Получение данных и вход student2

На pc1 под локальным пользователем:

```bash
getent passwd student2@lab.test
id student2@lab.test
getent group lab-students@lab.test
```

Получены UID student2 1164001106, основная группа domain users с GID 1164000513, lab-students с GID 1164001107 и участник student2 в записи группы.

Завершить графический сеанс, выбрать «Нет в списке?» и войти как student2@lab.test с его доменным паролем. Повторить на pc2. В каждом пользовательском сеансе без su -:

```bash
hostname -f
whoami
id
echo "$HOME"
klist
systemctl is-active sssd
```

Ожидаются имя соответствующего узла, student2, его UID и lab-students, /home/LAB.TEST/student2, TGT для student2@LAB.TEST и SSSD active. Все эти проверки успешно выполнены 9 октября на обоих клиентах до изменения формата имён в E09. После E09 на pc2 SSSD возвращает полные доменные имена, например student2@LAB.TEST; числовые UID и GID сохраняются. Новый графический вход student2 после изменения формата подтверждён 10 октября: на pc2 используется student2@LAB.TEST, сохранены UID, доменные и обычные локальные группы; после исправления NTP получен новый TGT.

Числовые идентификаторы относятся к этому домену и могут отличаться на новом стенде. [Результаты E02](../evidence/09-10-2026/README.md).

## 21. Сохранение результатов E02 и E09

Пользователь сообщил о создании снимков `after-e02-2026-10-09` и затем `after-e09-2026-10-09` на dc1, pc1 и pc2. Это подтверждение выполненного действия пользователем; изображений менеджера новых снимков в текущем архиве нет. Восстановление не проверялось. [Сведения о точках сохранения](../evidence/09-10-2026/vm-snapshots.txt).

Для новой точки: штатно завершить работу клиентов, затем dc1; в VirtualBox сделать снимок каждой ВМ с общим названием этапа. Перед новым опытом сохранить исходное состояние, после успешной проверки — отдельную итоговую точку. Снимок после этапа не следует использовать для записи изменений следующего опыта.

## 22. Текущая конфигурация pc2 после E09

[Протокол E09](../experiments/e09-local-domain-name-collision.md) содержит подготовку тёзок, сбой, выполненные команды исправления и исходные подтверждения. Создавать e09same и повторно вставлять параметр сейчас не требуется: записи и настройка уже существуют.

В `/etc/sssd/sssd.conf` на pc2:

```ini
[domain/LAB.TEST]
use_fully_qualified_names = true
```

Остальные параметры существующего раздела сохраняются. В ролях pc2 добавлена связь:

```text
domain users@LAB.TEST:users
```

Она назначает обычную роль users доменной группе с полным именем. Для воспроизведения на новом стенде механизм и ожидаемый набор групп следует сверить с [документацией ALT](https://docs.altlinux.org/ru-RU/alt-workstation/11.2/html/alt-workstation/activedirectory-rolelst.html). Изменение на pc1 в ходе E09 не выполнялось.

Проверка текущих источников из локального административного сеанса:

```bash
getent -s files passwd e09same
getent -s sss passwd e09same@lab.test
getent passwd e09same
getent passwd e09same@lab.test
id e09same
id e09same@lab.test
id student2@lab.test
```

На этом стенде ожидаются локальная запись с UID 1001 и доменная с UID 1164001108. У локального тёзки нет доменных групп; у доменного есть domain users@LAB.TEST и обычные группы роли users. student2 сохраняет lab-students@LAB.TEST (GID 1164001107).

Для проверки входа каждый раз завершить предыдущий графический сеанс. На pc2 через «Нет в списке?» использовать `e09same@lab.test` для доменного пользователя; локальная запись — `e09same` / «E09 local account». В новом пользовательском терминале без su:

```bash
id
echo "$HOME"
stat -c '%u:%g %n' "$HOME"
printf 'KRB5CCNAME=%s\n' "$KRB5CCNAME"
klist
loginctl list-sessions
```

Итоговая проверка E09 подтвердила правильные числовые идентификаторы и домашние каталоги; TGT присутствует только в доменном сеансе. Подпись карточки GDM сама по себе не подтверждает источник учётной записи.

Перед изменениями на pc2 сохранены `/root/sssd.conf.e09-before-fq-2026-10-09` и `/root/roles-e09-before-update-2026-10-09.tar.gz`. Их восстановление не выполнялось; в репозиторий эти конфигурационные копии не включены.

## 23. Синхронизация времени от dc1

10 октября установлено: Chrony на клиентах запущен, но не синхронизирован; dc1 получал внешнее время и запрещал NTP-доступ обоим клиентам. Полный разбор и исходные данные находятся в [time-synchronization.md](time-synchronization.md).

На новом стенде сначала проверить состояние dc1 и клиентский доступ:

```bash
chronyc tracking
chronyc -n sources -v
chronyc accheck 10.77.0.21
chronyc accheck 10.77.0.22
ss -lunp 'sport = :123'
```

Для сети этого стенда в /etc/chrony.conf dc1 используется:

```text
allow 10.77.0.0/24
```

Перед правкой предусмотрена резервная копия. Затем перезапустить chronyd, проверить active, Access allowed, UDP 123 и Normal на контроллере. Клиенты сохраняют источник 10.77.0.10.

На клиентах после исправления доступа были выполнены restart chronyd, burst 4/4, проверка выбора ^* 10.77.0.10 и затем makestep для остаточного смещения. При исправном времени повторять makestep не требуется. Обязательно проверить chronyc tracking и timedatectl status после следующего измерения: в выполненной работе yes появился не в первом запросе.

Результат: pc1 и pc2 синхронизированы именно с dc1; источник больше не остаётся неизвестным. [Исходные подтверждения](../evidence/10-10-2026/README.md).

## 24. Общий фон и блокировки — часть E05

Подробный воспроизводимый протокол находится в [experiments/e05-desktop-wallpaper.md](../experiments/e05-desktop-wallpaper.md). Краткий порядок для повторения:

1. Запустить dc1, затем pc1 и pc2; проверить DNS/SRV, SSSD, время и версии компонентов. На pc1 10 октября получены GPUI 0.2.55-alt1 и admx-basealt 0.8.0-alt1, на обоих клиентах gpupdate 0.16.1-alt1; профиль workstation, backend samba и system-policy=gpupdate.
2. Сохранить исходное состояние всех трёх ВМ. В выполненном опыте используется before-e05-2026-10-10.
3. Из локального административного сеанса pc1 открыть ADMC с доменной учётной записью администратора. Создать OU Lab-Computers, переместить туда PC1/PC2. Контроллер DC1 не перемещать.
4. Создать E05-Lab-Desktop со связью на Lab-Computers. Через «Изменить…» открыть GPUI; использовать компьютерную часть → Административные шаблоны → Система ALT → Настройки GNOME → Внешний вид → Фон рабочего стола.
5. Включить настройку, задать один URI обеим темам, сначала оставить обе блокировки снятыми. При необходимости прокрутить панель вниз к ОК.

URI для обоих полей:

```text
file:///usr/share/backgrounds/alt-gnome-desktop/alt-staraya-ladoga.jpg
```

На каждом клиенте из локальной административной учётки:

```bash
su -
gpupdate --target Computer --force
echo "gpupdate exit code: $?"
exit
```

В новых сеансах student2@lab.test без root проверить:

```bash
hostname -f
gsettings get org.gnome.desktop.background picture-uri
gsettings get org.gnome.desktop.background picture-uri-dark
gsettings writable org.gnome.desktop.background picture-uri
gsettings writable org.gnome.desktop.background picture-uri-dark
```

Без блокировок ожидаются true; попробовать записать пользовательские значения Эльбруса. Затем включить обе блокировки в GPUI, сохранить, обновить политики, перезагрузить клиентов и войти заново. Ожидаются Старая Ладога, false и отказ обеих попыток записи.

Для обратной проверки снять обе галочки, сохранив политику включённой. Обновить её, перезагрузить клиентов и войти заново. В выполненном опыте получены true и успешная запись обеих тем:

```bash
gsettings set org.gnome.desktop.background picture-uri "'file:///usr/share/backgrounds/alt-gnome-desktop/alt-elbrus.jpg'"
gsettings set org.gnome.desktop.background picture-uri-dark "'file:///usr/share/backgrounds/alt-gnome-desktop/alt-elbrus.jpg'"
gsettings get org.gnome.desktop.background picture-uri
gsettings get org.gnome.desktop.background picture-uri-dark
```

Нумерацию пунктов не включать в копируемые команды. Проверки пользовательских значений выполнять в графическом сеансе самого student2. Административную авторизацию ADMC выполнять из локального административного сеанса, чтобы не подменять пользовательский билет.

После этапа политика остаётся включённой, обе блокировки сняты, пользовательский фон — Эльбрус. Итоговая точка — after-e05-wallpaper-2026-10-10; [сведения о снимках](../evidence/10-10-2026/vm-snapshots.txt).

## 25. Следующий этап

* Продолжить E05 отдельной проверкой съёмных носителей на двух клиентах: исходный доступ, ограничение, отказ и отмена.
* E03, E04, E06-E08 остаются проверками по [плану исследования](research.md).
* После запуска ВМ проверять выбранный источник NTP; при повторении soft lockup сохранить расширенные данные.
* Перед новым опытом сохранить отдельные снимки; учитывать полные имена на pc2 и короткие на pc1.

Заметки об интерфейсе и диагностике ведутся в [журнале пользовательского опыта](user-experience.md).

## Источники

Основа инструкции составлена 28 сентября; разделы о клиентах дополнены 3 октября, DNS, ADMC/E02 и E09 — 9 октября, E05 и NTP — 10 октября по сохранённым результатам. Ссылки на конкретные разделы приведены рядом с соответствующими действиями. Полная повторная установка по обновлённой инструкции не проводилась.

- [Руководство Альт Домен 11.1](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/index.html).
- [Создание домена со встроенным DNS Samba](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/ch10s06s04.html).
- [Сети VirtualBox 7.2](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html).
- [Снимки VirtualBox 7.2](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/working-with-vms.html).
- [Полные имена SSSD](https://sssd.io/docs/ad/ad-provider.html).
- [Канонизация пользователя при входе](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/dm-login.html).
- [Группы и роли ALT Workstation](https://docs.altlinux.org/ru-RU/alt-workstation/11.2/html/alt-workstation/activedirectory-rolelst.html).
- [ALT: настройки gsettings и блокировки](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/gsettings.html).
- [ALT: gpupdate](https://docs.altlinux.org/ru-RU/alt-domain/11.1/html/alt-domain/gpupdate.html).
- [Chrony: конфигурация](https://chrony-project.org/doc/4.6/chrony.conf.html).
- [Chrony: команды диагностики](https://chrony-project.org/doc/4.6/chronyc.html).
