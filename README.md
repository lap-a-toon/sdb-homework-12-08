# Домашнее задание к занятию 12.8. «Резервное копирование баз данных» - Лабазов Александр


---

### Задание 1. Резервное копирование

### Кейс
Финансовая компания решила увеличить надёжность работы баз данных и их резервного копирования. 
```text
Берём во внимание, что это именно финансовая компания, и делаем допущение, что работает она круглосуточно во всех часовых поясах, скажем, РФ.
При этом ориентируемся на следующее:
- потребность в активности сервисов составляет около 16 часов (с 8:00 до 00:00)
- имеем объединенную базу для всех регионов
```
Необходимо описать, какие варианты резервного копирования подходят в случаях: 

1.1. *Необходимо восстанавливать данные в полном объёме за предыдущий день.*
```text
Следует выполнять полное холодное резервное копирование со снапшота в районе 01:00 (МСК), т.к. в указанное время наименьшая нагрузка на сервис, соответственно высвобождаются ресурсы для проведения бэкапа.
```
1.2. *Необходимо восстанавливать данные за час до предполагаемой поломки.*
```text
Следует настроить инкрементное или дифференциальное копирование на каждые, скажем 30 минут (я бы, при наличии ресурсов, склонился к дифференциальному на отдельном сервере).
Таким образом мы получим возможность откатить базу на состояние "за час до инцидента"
При этом хотелось бы не отменяя полного резервирования на предыдущие сутки, как минимум на протяжении недели.
```

1.3.* *Возможен ли кейс, когда при поломке базы происходило моментальное переключение на работающую или починенную базу данных.*
```text
Возможен, но это уже не резервкое копирование/восстановление, а реплицирование и это правильнее назвать High Availability, а не резервное копирование, потому как, в случае "поломки" самой базы, мы получим точную копию "поломанной" базы.
Данный кей пригоден для случаем, когда возникают проблемы с доступностью сервера (выход из строя железа, сети и т.д.)
```

---

### Задание 2. PostgreSQL

2.1. *С помощью официальной документации приведите пример команды резервирования данных и восстановления БД (pgdump/pgrestore).*

Довольно неплохо и понятно расписано резервное копирование [здесь][https://selectel.ru/blog/postgresql-backup-tools/]

Я бы предпочел делать полную копию так:
```sh
# pg_dumpall > /some_mounted_by_nfs_directory/dump_full.bak
```

Ну и восстановление, соответственно:
```sh
# pg_restore -Fc /some_mounted_by_nfs_directory/dump_full.bak
```

Если же речь идёт о резервировании какой-то конкретной БД, то резервирование:
```sh
# pg_dump -Ft my_database > /some_mounted_by_nfs_directory/my_database.tar
```

Восстановление:
```sh
# pg_restore -Ft my_database /some_mounted_by_nfs_directory/my_database.tar
```
или
```sh
# psql -U pg_user -W pg_password my_database < /some_mounted_by_nfs_directory/my_database.tar
```


2.1.* *Возможно ли автоматизировать этот процесс? Если да, то как?*
```text
Возможно.
Самый простой вариант, который с ходу приходит в голову -  bash-скрипт, запущенный по Cron-задаче, с выполнением соотвевтствующих записей в отдельный лог, а то и "рапортом" в мессенджер о старте и окончании операции.
```

---

### Задание 3. MySQL

3.1. *С помощью официальной документации приведите пример команды инкрементного резервного копирования базы данных MySQL.*
На мой взгляд такой командой вполне реально создать инкрементный бэкап (тесты не проводил):
```sh
mysqlbackup --incremental --incremental-base=history:last_backup \
  --backup-dir=/some_mounted_by_nfs_directory/2022-01-15 \
  --backup-image=increment_image.bi \
  backup-to-image
```

Ну и восстановление, соответственно:
```sh
mysqlbackup --backup-dir=/some_mounted_by_nfs_directory/2022-01-15 apply-log
```

3.1.* В каких случаях использование реплики будет давать преимущество по сравнению с обычным резервным копированием?
```text
Реплика будет полезна в случае необходимости обеспечения высокой доступности сервера.
Ориентируясь на условия, которые я описал в начале выполнения ДЗ - уместно было бы наспределить сервера-реплики по регионам, таким образом серверы БД будут как минимум быстрее обрабатывать запросы типа SELECT, что, соответственно, будет большим плюсом.
```

---
