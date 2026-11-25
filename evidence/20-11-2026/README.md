# Подтверждения за 20 ноября 2026 года

Работа возобновлена после перерыва. Подтверждены восстановление внешнего DNS и синхронизации времени, свежие доменные входы и завершение E05 в части съёмного носителя. Скриншоты переименованы по содержанию; текстовые файлы дополняют их выводами команд и пояснениями.

* [Протокол E05 со съёмным носителем](../../experiments/e05-removable-storage.md).
* [Диагностика DNS и времени](../../docs/time-synchronization.md#повторная-диагностика-13-октября-и-восстановление-20-ноября).
* [Записи в журнале](../../docs/journal.md#20-ноября-2026-года).

## Скриншоты

Файл                                                                     | Что подтверждает
-------------------------------------------------------------------------|-------------------------------------------------------
[01-dc1-resumed-console.png](01-dc1-resumed-console.png)                 | На экране dc1 после возврата осталась консоль октябрьской диагностики: NTP слушает UDP 123, Samba возвращает доменную SRV-запись, внешний запрос через неё не проходит.
[02-dc1-dns-ntp-before.png](02-dc1-dns-ntp-before.png)                   | Свежая проверка dc1: дата гостя 13 октября, внешний DNS через Samba не отвечает, у Chrony неизвестные адреса источников и Not synchronised.
[03-dc1-forwarder-direct-check.png](03-dc1-forwarder-direct-check.png)   | Прямой запрос pool.ntp.org: тайм-аут на 172.17.2.11; на 1.1.1.1 получены адреса. |
[04-virtualbox-dc1-reboot-error.png](04-virtualbox-dc1-reboot-error.png) | Диалог Windows об аварии VirtualBoxVM.exe во время перезагрузки dc1.
[05-dc1-dns-ntp-after.png](05-dc1-dns-ntp-after.png)                     | После повторного запуска dc1: дата 20 ноября, Samba/Chrony active, forwarder 1.1.1.1, оба DNS-запроса успешны, у Chrony есть выбранный источник и Normal.
[06-host-usb-copy-in-progress.png](06-host-usb-copy-in-progress.png)     | Копирование данных флешки на хосте: промежуточное состояние, 4%. Кадр не подтверждает завершение копирования.
[07-pc1-virtualbox-usb-menu.png](07-pc1-virtualbox-usb-menu.png)         | Меню передачи USB в VirtualBox: носитель USB Disk 2.0 [0200]; в заголовке pc1 видна точка before-tests-2026-10-13.
[08-pc1-usb-before-policy.png](08-pc1-usb-before-policy.png)             | До запрета на pc1 видны смонтированный том и тестовый файл; на этом кадре файл ещё пуст.
[09-pc2-usb-before-policy.png](09-pc2-usb-before-policy.png)             | До запрета на pc2 открыт e05-test.txt со строкой E05 USB test.
[10-admc-lab-computers-ou.png](10-admc-lab-computers-ou.png)             | В ADMC видны Lab и Lab-Computers; сеанс administrator@lab.test. Этот кадр показывает подразделение, а не ссылку новой GPO.
[11-gpui-gnome-templates.png](11-gpui-gnome-templates.png)               | В GPUI открыт объект E05-Lab-USB; в показанной части дерева видна ветка настроек GNOME.
[12-gpui-alt-templates.png](12-gpui-alt-templates.png)                   | В том же объекте показана ветка «Система ALT» во время поиска настройки съёмных носителей.
[13-gpui-usb-deny-enabled.png](13-gpui-usb-deny-enabled.png)             | Параметр запрета для всех классов съёмных запоминающих устройств в состоянии «Включено», GPUI сообщил о сохранении.
[14-pc1-usb-mount-denied.png](14-pc1-usb-mount-denied.png)               | При запрете «Файлы» на pc1 видят том, но показывают Not authorized to perform operation.
[15-pc2-usb-mount-denied.png](15-pc2-usb-mount-denied.png)               | При запрете «Файлы» на pc2 показывают такой же отказ. |
[16-gpui-usb-deny-disabled.png](16-gpui-usb-deny-disabled.png)           | В той же настройке запрета выбрано и сохранено «Отключено». |
[17-pc1-usb-access-restored.png](17-pc1-usb-access-restored.png)         | После снятия запрета pc1: с флешки снова открыт e05-test.txt и видна тестовая строка.
[18-pc2-usb-access-restored.png](18-pc2-usb-access-restored.png)         | После снятия запрета pc2: с флешки открыт тот же файл, в пути отображается полное доменное имя.

## Выводы команд и сохранение состояния

Файл                                                           | Содержание
---------------------------------------------------------------|-----------------------------------------------------------------
[dc1-dns-ntp-checks.txt](dc1-dns-ntp-checks.txt)               | Выбранные строки с кадров dc1, проверка внешнего DNS до и после изменения, авария и повторный запуск ВМ.
[clients-time-and-student2.txt](clients-time-and-student2.txt) | Синхронизация pc1 и pc2 с dc1, UID/GID и свежие TGT после восстановления входа.
[pc1-admx-templates.txt](pc1-admx-templates.txt)               | Путь к шаблону RemovableStorage и локализациям на pc1.
[gpupdate-computer-runs.txt](gpupdate-computer-runs.txt)       | Выводы четырёх запусков gpupdate: включение и отключение запрета на двух клиентах.
[usb-test-stages.txt](usb-test-stages.txt)                     | Исходное монтирование, оба отказа UDisks и восстановленные точки монтирования.
[vm-snapshots.txt](vm-snapshots.txt)                           | Какие снимки подтверждены и какие были предложены.

На обеих ВМ при запрете устройство оставалось видимым, но не монтировалось. После отключения запрета тот же файл читался с флешки на обоих клиентах. Причина аварии VirtualBox не установлена; здесь сохранён диалог Windows, без журнала аварии.
