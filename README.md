# Сурмач Андрей Андреевич
# Backup_

# Задание_1

Необходимо восстанавливать данные в полном объёме за предыдущий день
В таком случае можно делать полный бэкап каждый день – восстановление пройдёт быстрее и проблем с таким бэкапом быть не должно. Разве что место занимает, но если другие дни не требуются, одну полную копию можно пережить.

Необходимо восстанавливать данные за час до предполагаемой поломки.
Каждый час делать полную копию накладно. И даже дифференциальную. Надо что-то, что будет быстро создавать копии, не сильно долго этим заниматься и занимать поменьше места.

В таком случае подойдут инкрементные бэкапы, записывающие изменения с момента последнего бэкапа.

Возможен ли кейс, когда при поломке базы происходило моментальное переключение на работающую или починенную базу данных.
Если проблема с доступностью, то спасти может всё, что создаёт какую-никакую копию данных – репликация (нагрузку примут оставшиеся slave или master), RAID, DRBD, SAN-кластера.

Однако же, если что-то сломало саму базу данных, это могло произойти на всех копиях, и тут как будто только откатываться до последнего сохранённого бэкапа.

# Задание_2

Резервирование и восстановление данных в PostgreSQL
Выгрузка в виде скрипта (plain)

```
pg_dump <database_name> > <script_name>.sql
```
или то же самое
```
pg_dump -Fp <database_name> > <script_name>.sql
```
Выгрузка в виде специального файла (custom):
```
pg_dump -Fc <database_name> > <file_name>.dump
```
Выгрузка в виде каталога:
```
pg_dump -Fd <database_name> -f <dirname>
```
Выгрузка в виде tar-архива:
```
pg_dump -Ft <database_name> -f <tar_name>.tar
```
Все указанные форматы пригодны для восстановления через pg_restore.
Восстановление:
```
pg_restore -d <database_name> <file_name>
```
При этом база данных уже должна быть создана, но быть пустой.

Автоматизация резервирования
Подходит любая утилита, позволяющая задать время исполнения.

средствами Linux это можно сделать с помощью crontab;
конкретно для PosgreSQL можно использовать pgAgent – его потребуется установить в качестве расширения, запустить как демона и задать расписание.

# Задание_3
Инкрементное резервное копирование БД MySQL
MySQL Enterprise просто предлагает использовать инструмент mysqlbackup; тогда команда будет выглядеть как:
```
mysqlbackup --defaults-file=/home/dbadmin/my.cnf \
  --incremental --incremental-base=history:last_backup \
  --backup-dir=<path_to_backup> \
  --backup-image=<image_file_name>.bi \
   backup-to-image
```
Если же мы использовать mysqlbackup не можем и имеем под рукой только mysqldump, то алгоритм становится сложнее. Он предполагает использование binary logs – логов о событиях в БД.

Сначала делаем полный бэкап, и затем записываем инкрементные изменения через фиксацию логов.
```
mysqldump -uroot -p --all-databases --single-transaction --flush-logs --master-data=2 > <full_backup_name>.sql
```
Здесь ключевой параметр – --flush-logs, который закрывает запись логов в старый файл и открывает новый. При создание следующего "инкремента" нужно будет также "закрыть" новый файл и "открыть" новый.
```
mysqladmin -uroot -p flush-logs
```
При восстановлении данных понадобится также сначала восстановить изначальный полный бэкап, созданный с помощью mysqldump:
```
mysql -u root -p mydb < <full_backup_name>.sql
```
Затем последовательно применить все binary logs:
```
mysqlbinlog /var/log/mysql/mysql-bin.<number_of_log> | mysql -uroot -p mydb
```
Когда лучше реплика, чем резервное копирование
Если проблема не в том, что какие-то данные поломали таблицы или базу данных целиком, а в доступе к данным, реплика позволит достаточно быстро переключить соединения; потеря данных при этом будет минимальной.

