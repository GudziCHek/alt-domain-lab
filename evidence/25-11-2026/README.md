# Подтверждения за 25 ноября 2026 года

Проверка стенда и [E03: минимальная длина доменного пароля](../../experiments/e03-domain-password-policy.md). Время — MSK (UTC+3). На трёх кадрах с формами закрыты оба парольных поля; остальные кадры сохранены без изменений.

Файл | Что показывает
-----|--------------
[01-dc1-e03-baseline.png](01-dc1-e03-baseline.png) | На dc1 работают Samba, Chrony и внешний DNS; общий минимум пароля — 7.
[02-admc-e03-create-user-redacted.png](02-admc-e03-create-user-redacted.png) | Форма создания тестовой учётной записи `e03test`.
[03-admc-e03-six-character-attempt-redacted.png](03-admc-e03-six-character-attempt-redacted.png) | Подготовлена первая попытка сброса пароля уже созданной записи.
[04-admc-e03-baseline-reset-denied.png](04-admc-e03-baseline-reset-denied.png) | Ответ ADMC на первую попытку: «Пароль не соответствует правилам».
[05-admc-e03-seven-character-attempt-redacted.png](05-admc-e03-seven-character-attempt-redacted.png) | Вторая попытка сброса; результат подтверждён проверкой `kinit`.
[06-dc1-e03-policy-applied.png](06-dc1-e03-policy-applied.png) | `E03-Min12` назначена напрямую `e03test`; общий минимум остаётся 7.
[07-admc-e03-policy-reset-denied.png](07-admc-e03-policy-reset-denied.png) | Общий текст отказа при попытке установить пароль из 11 символов. Длина записана по ходу опыта, окно ошибки закрывает парольное поле.
[08-dc1-e03-policy-removed.png](08-dc1-e03-policy-removed.png) | PSO отвязана и удалена, общий минимум домена — 7.
[pc1-e03-baseline.txt](pc1-e03-baseline.txt) | Время, выбранный источник NTP, Kerberos SRV и доменная запись на pc1.
[pc1-e03-kinit.txt](pc1-e03-kinit.txt) | Четыре проверки получения начального билета с кодом `0`.
[dc1-e03-password-policy.txt](dc1-e03-password-policy.txt) | Параметры исходной политики, прямое назначение PSO и её удаление.
[vm-snapshots.txt](vm-snapshots.txt) | Подтверждение выключения и итоговых снимков dc1 и pc1.
