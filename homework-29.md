  Развернуть CitusDB в GKE, залить 10 Гб чикагского такси. Шардировать. Оценить производительность. Описать проблемы, с которыми столкнулись
    
   Создадим кластер из одной ноды (пока что)
   
    gcloud container clusters create homework-29 --num-nodes "1" --preemptible
    kubectl create -f secrets.yaml
    kubectl create -f master.yaml
    kubectl create -f workers.yaml
    PS C:\TEMP\pgGKE\citus> kubectl get pods
    NAME                            READY   STATUS    RESTARTS   AGE
    citus-master-78ff549b8f-bgfpm   1/1     Running   0          3m7s
    citus-worker-0                  1/1     Running   0          81s
    citus-worker-1                  1/1     Running   0          67s
    citus-worker-2                  1/1     Running   0          45s
    PS C:\TEMP\pgGKE\citus> kubectl exec -it pod/citus-master-78ff549b8f-bgfpm -- bash
    root@citus-master-78ff549b8f-bgfpm:/# psql -U postgres
    psql (10.3 (Debian 10.3-1.pgdg90+1))
    Type "help" for help.

    postgres=# SELECT * FROM master_get_active_worker_nodes();
              node_name           | node_port
    ------------------------------+-----------
     citus-worker-2.citus-workers |      5432
     citus-worker-0.citus-workers |      5432
     citus-worker-1.citus-workers |      5432
    (3 rows)

  Поправим селектор, чтобы сработал доступ через loadbalancer

    kubectl create -f ..\postgres-service2.yaml
    kubectl edit service postgres
      selector:
        app: citus-master

    PS C:\TEMP\pgGKE\citus> kubectl get service
    NAME            TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)          AGE
    citus-master    NodePort       10.32.2.26    <none>          5432:32260/TCP   22m
    citus-workers   ClusterIP      None          <none>          5432/TCP         20m
    kubernetes      ClusterIP      10.32.0.1     <none>          443/TCP          129m
    postgres        LoadBalancer   10.32.7.144   35.232.146.25   5432:30292/TCP   12m

    psql -U postgres -h 35.232.146.25

    postgres=# SELECT * FROM master_get_active_worker_nodes();
              node_name           | node_port
    ------------------------------+-----------
     citus-worker-2.citus-workers |      5432
     citus-worker-0.citus-workers |      5432
     citus-worker-1.citus-workers |      5432
    (3 rows)

    CREATE TABLE taxi_trips(unique_key text
    ,taxi_id text
    ,trip_start_timestamp timestamp
    ,trip_end_timestamp timestamp
    ,trip_seconds bigint
    ,trip_miles float
    ,pickup_census_tract bigint
    ,dropoff_census_tract bigint
    ,pickup_community_area bigint
    ,dropoff_community_area bigint
    ,fare float
    ,tips float
    ,tolls float
    ,extras float
    ,trip_total float
    ,payment_type text
    ,company text
    ,pickup_latitude float
    ,pickup_longitude float
    ,pickup_location text
    ,dropoff_latitude float
    ,dropoff_longitude float
    ,dropoff_location text);


    SELECT create_distributed_table('taxi_trips', 'unique_key');

    postgres=# SELECT * FROM pg_dist_shard;
     logicalrelid | shardid | shardstorage | shardminvalue | shardmaxvalue
    --------------+---------+--------------+---------------+---------------
     taxi_trips   |  102008 | t            | -2147483648   | -2013265921
     taxi_trips   |  102009 | t            | -2013265920   | -1879048193
     taxi_trips   |  102010 | t            | -1879048192   | -1744830465
     taxi_trips   |  102011 | t            | -1744830464   | -1610612737
     taxi_trips   |  102012 | t            | -1610612736   | -1476395009
     taxi_trips   |  102013 | t            | -1476395008   | -1342177281
     taxi_trips   |  102014 | t            | -1342177280   | -1207959553
     taxi_trips   |  102015 | t            | -1207959552   | -1073741825
     taxi_trips   |  102016 | t            | -1073741824   | -939524097
     taxi_trips   |  102017 | t            | -939524096    | -805306369
     taxi_trips   |  102018 | t            | -805306368    | -671088641
     taxi_trips   |  102019 | t            | -671088640    | -536870913
     taxi_trips   |  102020 | t            | -536870912    | -402653185
     taxi_trips   |  102021 | t            | -402653184    | -268435457
     taxi_trips   |  102022 | t            | -268435456    | -134217729
     taxi_trips   |  102023 | t            | -134217728    | -1
     taxi_trips   |  102024 | t            | 0             | 134217727
     taxi_trips   |  102025 | t            | 134217728     | 268435455
     taxi_trips   |  102026 | t            | 268435456     | 402653183
     taxi_trips   |  102027 | t            | 402653184     | 536870911
     taxi_trips   |  102028 | t            | 536870912     | 671088639
     taxi_trips   |  102029 | t            | 671088640     | 805306367
     taxi_trips   |  102030 | t            | 805306368     | 939524095
     taxi_trips   |  102031 | t            | 939524096     | 1073741823
     taxi_trips   |  102032 | t            | 1073741824    | 1207959551
     taxi_trips   |  102033 | t            | 1207959552    | 1342177279
     taxi_trips   |  102034 | t            | 1342177280    | 1476395007
     taxi_trips   |  102035 | t            | 1476395008    | 1610612735
     taxi_trips   |  102036 | t            | 1610612736    | 1744830463
     taxi_trips   |  102037 | t            | 1744830464    | 1879048191
     taxi_trips   |  102038 | t            | 1879048192    | 2013265919
     taxi_trips   |  102039 | t            | 2013265920    | 2147483647
    (32 rows)

  Создадим ещё одну ВМ с постгрес-клиентом. Сюда поставим gcsfuse и будем заливать Чикакгское такси 10ГБ.
  
    gcloud compute instances create homework-29-psql

    postgres@homework-29-psql:/$ gcsfuse chicago-trips /mnt/taxi

    for F in /mnt/taxi/file-0000000000{00..30}.csv; do echo "Proccessing $F .."; psql -U postgres -h 34.67.171.89 -c "\copy taxi_trips from '$F' CSV HEADER"; done

    postgres=# SELECT logicalrelid AS name,pg_size_pretty(citus_table_size(logicalrelid)) AS size FROM pg_dist_partition;
        name    |  size
    ------------+---------
     taxi_trips | 8655 MB
    (1 row)


    postgres=# \timing
    Timing is on.
    postgres=# SELECT payment_type, round(sum(tips)/sum(tips+fare)*100) tips_persent, count(*)
    postgres-# FROM taxi_trips
    postgres-# group by payment_type
    postgres-# order by 3 desc;
     payment_type | tips_persent |  count
    --------------+--------------+----------
     Cash         |            0 | 11641728
     Credit Card  |           18 |  8377018
     No Charge    |            3 |   103362
     Unknown      |            4 |    38877
     Dispute      |            0 |    11781
     Prcard       |            2 |     8110
     Pcard        |            3 |     4475
     Mobile       |           16 |     2851
     Way2ride     |           16 |       92
     Prepaid      |            0 |        7
    (10 rows)

    Time: 712923.068 ms (11:52.923)
    
  Почти 12 минут. Прибавим ноды.

    gcloud container clusters resize homework-29 --num-nodes=3

    postgres=# SELECT payment_type, round(sum(tips)/sum(tips+fare)*100) tips_persent, count(*)
    FROM taxi_trips
    group by payment_type
    order by 3 desc;
     payment_type | tips_persent |  count
    --------------+--------------+----------
     Cash         |            0 | 11641728
     Credit Card  |           18 |  8377018
     No Charge    |            3 |   103362
     Unknown      |            4 |    38877
     Dispute      |            0 |    11781
     Prcard       |            2 |     8110
     Pcard        |            3 |     4475
     Mobile       |           16 |     2851
     Way2ride     |           16 |       92
     Prepaid      |            0 |        7
    (10 rows)

    Time: 142473.971 ms (02:22.474)

Стало 2 минуты.
