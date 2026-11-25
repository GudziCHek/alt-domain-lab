# Подтверждения за 24 ноября 2026 года

Проверка стенда после запуска и [эксперимент E04](../../experiments/e04-domain-account-disable.md). Время в системных выводах и на экране клиентов — MSK (UTC+3). PNG сохранены в исходном виде; текстовые файлы отмечают, откуда взяты выводы.

| Файл | Что показывает |
| --- | --- |
| [01-dc1-baseline.png](01-dc1-baseline.png) | На dc1 активны Samba и Chrony, время синхронизировано, локальный DNS отвечает для доменной SRV-записи и внешнего имени. |
| [02-admc-e04-create-form.png](02-admc-e04-create-form.png) | Форма создания пользователя: имя задано, поле имени для входа ещё пустое. |
| [03-admc-e04-create-filled.png](03-admc-e04-create-filled.png) | Заполнено имя для входа `e04test` в `lab.test`; доступна кнопка сохранения. |
| [04-pc2-e04-session-before-disable.png](04-pc2-e04-session-before-disable.png) | Исходный сеанс pc2, TGT с 23:37:44, PID оболочки `4827`. |
| [05-dc1-e04-account-enabled.png](05-dc1-e04-account-enabled.png) | Объект `e04test` в OU `Lab`, исходный `userAccountControl: 512`. |
| [06-admc-e04-account-disabled.png](06-admc-e04-account-disabled.png) | ADMC сообщает об отключении `e04test`. |
| [07-dc1-e04-account-disabled.png](07-dc1-e04-account-disabled.png) | На dc1 после отключения получено `userAccountControl: 514`. |
| [08-pc2-e04-original-session-after-disable.png](08-pc2-e04-original-session-after-disable.png) | Старый терминал pc2 отвечает в 23:45:18; PID `4827`, в кеше прежний TGT. |
| [09-pc1-e04-kinit-denied.png](09-pc1-e04-kinit-denied.png) | Контроллер отклонил запрос свежего TGT; код завершения `1`. |
| [10-pc1-e04-gdm-denied.png](10-pc1-e04-gdm-denied.png) | Новый графический вход на pc1 отклонён; GDM показывает общее сообщение о входе с паролем. |
| [11-dc1-e04-account-reenabled.png](11-dc1-e04-account-reenabled.png) | После обратного включения на dc1 снова `userAccountControl: 512`. |
| [12-dc1-poweroff-local-user.png](12-dc1-poweroff-local-user.png) | После опыта Bash не нашёл `poweroff`, а `sudo` отказал локальной учётке. |
| [client-baseline.txt](client-baseline.txt) | Присланные результаты проверки времени, DNS и поиска доменной учётки на клиентах. |
| [e04-client-sessions.txt](e04-client-sessions.txt) | Исходные и итоговые `id` и `klist` для обеих ВМ; отдельно — наблюдение в старом сеансе pc2. |
| [vm-snapshots.txt](vm-snapshots.txt) | Подтверждённые снимки до опыта и статус итоговых снимков. |
