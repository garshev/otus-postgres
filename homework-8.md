Домашнее задание
Механизм блокировок
Цель: 
понимать как работает механизм блокировок объектов и строк

Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

    postgres=# alter system set log_lock_waits TO on;
    ALTER SYSTEM
    postgres=# alter system set deadlock_timeout TO 200;
    ALTER SYSTEM
    postgres=# select pg_reload_conf();

    Начинаем 2 транзакции, в каждой выполняем 
    update t SET i = i + 1 where i = 1;
    В логе видим
    postgres@postgres LOG:  process 4097 acquired ShareLock on transaction 570 after 20591.209 ms

Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

    postgres=# SELECT pid,locktype, relation::REGCLASS, transactionid, mode, granted FROM pg_locks order by pid;
     pid  |   locktype    | relation | transactionid |       mode       | granted 
    ------+---------------+----------+---------------+------------------+---------
     4319 | tuple         | t        |               | ExclusiveLock    | f
     4319 | transactionid |          |           575 | ExclusiveLock    | t
     4319 | relation      | t        |               | RowExclusiveLock | t
     4319 | virtualxid    |          |               | ExclusiveLock    | t
     4349 | virtualxid    |          |               | ExclusiveLock    | t
     4349 | transactionid |          |           574 | ExclusiveLock    | t
     4349 | relation      | t        |               | RowExclusiveLock | t
     4349 | transactionid |          |           572 | ShareLock        | f
     4349 | tuple         | t        |               | ExclusiveLock    | t
     4372 | transactionid |          |           572 | ExclusiveLock    | t
     4372 | relation      | t        |               | RowExclusiveLock | t
     4372 | virtualxid    |          |               | ExclusiveLock    | t
     4397 | virtualxid    |          |               | ExclusiveLock    | t
     4397 | relation      | pg_locks |               | AccessShareLock  | t
    (14 rows)


    1я транзакция (pid 4372) получает RowExclusiveLock на таблицу и ExclusiveLock на id своей транзакции.
    2я транзакция (pid 4349) получает RowExclusiveLock на таблицу, ExclusiveLock на изменяемый кортеж, на свой id и пытается получить lock id первой транзакции.
    3я транзакция (pid 4319) уже не может получить ExclusiveLock на кортеж.

Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

    Да, в логе видно что тразакции ждут друг друга

    2021-10-31 15:51:20.226 UTC [4349] postgres@postgres DETAIL:  Process holding the lock: 4372. Wait queue: 4349.
    2021-10-31 15:51:20.226 UTC [4349] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "t"
    2021-10-31 15:51:20.226 UTC [4349] postgres@postgres STATEMENT:  update t SET i = i + 1 where i = 1;
    2021-10-31 15:59:58.753 UTC [4319] postgres@postgres DETAIL:  Process holding the lock: 4349. Wait queue: 4319.
    2021-10-31 15:59:58.753 UTC [4319] postgres@postgres STATEMENT:  update t SET i = i + 1 where i = 1;
    2021-10-31 16:16:09.032 UTC [4349] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "t"
    2021-10-31 16:16:09.032 UTC [4349] postgres@postgres STATEMENT:  update t SET i = i + 1 where i = 1;
    2021-10-31 16:16:09.032 UTC [4319] postgres@postgres STATEMENT:  update t SET i = i + 1 where i = 1;
    2021-10-31 16:16:09.232 UTC [4319] postgres@postgres DETAIL:  Process holding the lock: 4349. Wait queue: 4319.
    2021-10-31 16:16:09.232 UTC [4319] postgres@postgres CONTEXT:  while locking tuple (0,5) in relation "t"
    2021-10-31 16:16:09.232 UTC [4319] postgres@postgres STATEMENT:  update t SET i = i + 1 where i = 1;


Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга? 
Попробуйте воспроизвести такую ситуацию.

    Судя по статье на хабре такое маловероятно, но можно сделать если транзакции будут обновлять строки в разном порядке. 

