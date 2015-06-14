Title: Проверка консистентности бэкапов
Date: 2015-06-06 20:00
Category: PostgreSQL
Tags: PostgreSQL, barman, backup, monitoring
Lang: ru
Slug: barman-backups-check

Однажды в результате наложения двух человеческих ошибок при выкатке кода
на базы мы сделали `DROP SCHEMA data CASCADE;` на всех шардах одного из
кластеров, в котором лежало около 3 ТБ данных. Это добавило нам много седых
волос, позволило проверить свои навыки в PITR прямо в бою и заставило
по-другому относиться к резервным копиям.

Та история закончилась хорошо. Инцидент случился ближе к концу рабочего дня,
когда нагрузка уже спадала, а к утру мы восстановили всё из бэкапов на нужный
момент времени. Резервные копии мы делали всегда и всегда мониторили тот факт,
что они делаются. А вот проверку того, сможем ли мы из них восстановиться, мы
в момент переезда на [barman](http://www.pgbarman.org) выкинули из-за
затратности.

Восстановление одного из шардов затянулось дольше остальных из-за того, что из
последнего бэкапа нам подняться не удалось, пришлось восстанавливаться из
предпоследнего (бэкапы мы делаем каждый день). Потому проверку консистентности
после факапа решили вернуть и в результате получилась пара скриптов, которые
можно посмотреть
[тут](https://github.com/dev1ant/misc/tree/master/backups_checking). Один из
них (`check_backup_consistency.py`) последовательно разворачивает последний
бэкап каждого кластера, запускает PostgreSQL с `recovery_target = 'immediate'`
и дожидается достижения базой консистентного состояния.

Второй (`check_xlogs.sh`) проверяет тот
факт, что на машине с бэкапами есть все необходимые WAL'ы (c первого WAL'а
первого бэкапа до последнего заархивированного). В общем случае archiver
гарантирует последовательную отправку WAL'ов в архив и если у вас правильно
указан `archive_command`, то проблем с этим быть не должно. Но у нас
бывали случаи, когда на разделе с `pg_xlog` заканчивалось место и мы меняли
`archive_command` на перекладывание WAL'ов локально. При очередной выкатке
состояния на базы `archive_command` возвращалась на место, а вот переложенные
локально WAL'ы до архива донести могли забыть.

Эти проверки мы запускаем по cron'у, а мониторинг смотрит на status-файлы,
которые скрипты пишут в `/tmp`. Бэкапы мы начинаем делать в 2:00 и последние
из них добегают около 6:00 (слава инкрементальным бэкапам в barman 1.4). И в
середине дня (около 15:00-16:00) мы уже знаем, можно ли снова делать `DROP
SCHEMA` :)

Возможно, кому-то эти скрипты пригодятся. Будут вопросы - обращайтесь.