# ДЗ: Работа c PostgreSQL в Kubernetes

### Развернуть CitusDB в GKE, залить 10 Гб чикагского такси. Шардировать. Оценить производительность. Описать проблемы, с которыми столкнулись

Сначала разворачиваем Kubernetes:
```
gcloud beta container 
    --project "postgres2021-26091996-338403" clusters create "cluster-1"
    --zone "us-central1-c"
    --no-enable-basic-auth
    --cluster-version "1.21.6-gke.1500"
    --release-channel "regular"
    --machine-type "e2-medium"
    --image-type "COS_CONTAINERD"
    --disk-type "pd-standard"
    --disk-size "100"
    --metadata disable-legacy-endpoints=true
    --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" 
    --max-pods-per-node "110"
    --num-nodes "3"
    --logging=SYSTEM,WORKLOAD
    --monitoring=SYSTEM
    --enable-ip-alias
    --network "projects/postgres2021-26091996-338403/global/networks/default"
    --subnetwork "projects/postgres2021-26091996-338403/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility
    --default-max-pods-per-node "110"
    --no-enable-master-authorized-networks
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver
    --enable-autoupgrade
    --enable-autorepair
    --max-surge-upgrade 1
    --max-unavailable-upgrade 0
    --enable-shielded-nodes
    --node-locations "us-central1-c"
```

Далее после скачивания файлов из репозитория разворачиваем citus. Тут сначала не хотел разворачиваться мастер, но минорные правки файла marter.yaml все решили (+ аналогичные правки внес в workers.yaml):
```
kubectl create -f secrets.yaml
kubectl create -f master.yaml
kubectl create -f workers.yaml
```

В результате всё поднялось и kubectl get all выдало:
```
NAME                                READY   STATUS    RESTARTS   AGE
pod/citus-master-78ff549b8f-qh6mb   1/1     Running   0          4m31s
pod/citus-worker-0                  1/1     Running   0          83s
pod/citus-worker-1                  1/1     Running   0          70s
pod/citus-worker-2                  1/1     Running   0          40s

NAME                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/citus-master    ClusterIP   None         <none>        5432/TCP   6m49s
service/citus-workers   ClusterIP   None         <none>        5432/TCP   84s
service/kubernetes      ClusterIP   10.8.0.1     <none>        443/TCP    105m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/citus-master   1/1     1            1           4m32s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/citus-master-78ff549b8f   1         1         1       4m32s

NAME                            READY   AGE
statefulset.apps/citus-worker   3/3     84s
```

Теперь нужно скачать в контейнер мастера файлы чикагского такси. Тут пришлось немного помучиться, т.к. не сразу понял, как во время лекции скачивался файл из Cloud Storage. По итогу разобрался и сделал свой бакет с такси публичным, после чего скачал необходимый объем файлов скриптом:
```
# coding=utf-8

import requests
import os

i = 0
j = 0
res = []
for i in range(0, 4):
    for j in range(0, 10):
        num = str(i) + str(j)
        res.append(num)

print(res)

for el in res:
    URL = 'https://storage.googleapis.com/hw-taxi/taxi0000000000' + el + '.csv'
    store = 'tmp/taxi/taxi0000000000' + el + '.csv'
    os.system('wget -O %s %s' % (store, URL))
```

После скачивания файлов я сначала добавил extension на мастер и воркеры:
```
create extension if not exists "uuid-ossp";
```

А затем создал на мастере таблицу и шардировал её по колонке taxi_id (сначала хотел по времениначала поездки, но судя по документации адекватное шардирование по времени появилось только в более поздних версиях Citus):
```
create table taxi_trips (
    unique_key text, 
    taxi_id text, 
    trip_start_timestamp TIMESTAMP, 
    trip_end_timestamp varchar, 
    trip_seconds varchar, 
    trip_miles varchar, 
    pickup_census_tract varchar, 
    dropoff_census_tract varchar, 
    pickup_community_area varchar, 
    dropoff_community_area varchar, 
    fare varchar, 
    tips varchar, 
    tolls varchar, 
    extras varchar, 
    trip_total varchar, 
    payment_type text, 
    company text, 
    pickup_latitude varchar, 
    pickup_longitude varchar, 
    pickup_location text, 
    dropoff_latitude varchar, 
    dropoff_longitude varchar, 
    dropoff_location text
);

select create_distributed_table('taxi_trips', 'taxi_id');
```

Для загрузки данных из csv в базу я воспользовался скриптом:
```
import os

i = 0
j = 0
res = []
for i in range(0, 4):
    for j in range(0, 10):
        num = str(i) + str(j)
        query = "copy taxi_trips(unique_key, \
                taxi_id, \
                trip_start_timestamp, \
                trip_end_timestamp, \
                trip_seconds, \
                trip_miles, \
                pickup_census_tract, \
                dropoff_census_tract, \
                pickup_community_area, \
                dropoff_community_area, \
                fare, \
                tips, \
                tolls, \
                extras, \
                trip_total, \
                payment_type, \
                company, \
                pickup_latitude, \
                pickup_longitude, \
                pickup_location, \
                dropoff_latitude, \
                dropoff_longitude, \
                dropoff_location) \
                from '/tmp/taxi/taxi0000000000%s.csv' DELIMITER ',' CSV HEADER;" % num

        os.system('psql -U postgres -c "' + query + '"')
```

Сравнение длительности выполнения запросов показало, что citus выполняет запросы почти в 2 раза медленнее, если запрос строится по всей таблице:
1) На сервере с Postgres
```
otus=# explain analyze select count(*) from taxi_trips group by taxi_id;
                                                                          QUERY PLAN

------------------------------------------------------------------------------------------------------------------------------
--------------------------------
 Finalize GroupAggregate  (cost=1585280.78..1586788.46 rows=5951 width=139) (actual time=53240.395..53265.245 rows=8199 loops=
1)
   Group Key: taxi_id
   ->  Gather Merge  (cost=1585280.78..1586669.44 rows=11902 width=139) (actual time=53240.380..53258.440 rows=24144 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=1584280.76..1584295.63 rows=5951 width=139) (actual time=53217.579..53219.448 rows=8048 loops=3)
               Sort Key: taxi_id
               Sort Method: quicksort  Memory: 2335kB
               Worker 0:  Sort Method: quicksort  Memory: 2324kB
               Worker 1:  Sort Method: quicksort  Memory: 2331kB
               ->  Partial HashAggregate  (cost=1583848.15..1583907.66 rows=5951 width=139) (actual time=53179.650..53186.674
rows=8048 loops=3)
                     Group Key: taxi_id
                     Batches: 1  Memory Usage: 2961kB
                     Worker 0:  Batches: 1  Memory Usage: 2961kB
                     Worker 1:  Batches: 1  Memory Usage: 2961kB
                     ->  Parallel Seq Scan on taxi_trips  (cost=0.00..1529553.77 rows=10858877 width=131) (actual time=0.683..
46951.114 rows=8681641 loops=3)
 Planning Time: 11.077 ms
 Execution Time: 53267.975 ms
(18 rows)
```
2) На сервере с Citus:
```
postgres=# explain analyze select count(*) from taxi_trips group by taxi_id;
                                                                                         QUERY PLAN

------------------------------------------------------------------------------------------------------------------------------------------------------------------------
---------------------
 HashAggregate  (cost=0.00..0.00 rows=0 width=0) (actual time=90775.097..90779.104 rows=8199 loops=1)
   Group Key: remote_scan.worker_column_2
   ->  Custom Scan (Citus Real-Time)  (cost=0.00..0.00 rows=0 width=0) (actual time=90766.762..90767.373 rows=8199 loops=1)
         Task Count: 32
         Tasks Shown: One of 32
         ->  Task
               Node: host=citus-worker-0.citus-workers port=5432 dbname=postgres
               ->  Finalize GroupAggregate  (cost=54466.51..54473.13 rows=265 width=140) (actual time=2251.677..2252.021 rows=280 loops=1)
                     Group Key: taxi_id
                     ->  Sort  (cost=54466.51..54467.83 rows=530 width=140) (actual time=2251.669..2251.742 rows=829 loops=1)
                           Sort Key: taxi_id
                           Sort Method: quicksort  Memory: 245kB
                           ->  Gather  (cost=54386.88..54442.53 rows=530 width=140) (actual time=2249.656..2249.956 rows=829 loops=1)
                                 Workers Planned: 2
                                 Workers Launched: 2
                                 ->  Partial HashAggregate  (cost=53386.88..53389.53 rows=265 width=140) (actual time=2244.145..2244.227 rows=276 loops=3)
                                       Group Key: taxi_id
                                       ->  Parallel Seq Scan on taxi_trips_102040 taxi_trips  (cost=0.00..51531.92 rows=370992 width=132) (actual time=15.669..2083.805
rows=295147 loops=3)
                   Planning time: 0.612 ms
                   Execution time: 2255.086 ms
 Planning time: 91.133 ms
 Execution time: 90780.517 ms
(22 rows)
```

Однако если выполнять запрос с ограничением по колонке, по которой шардировали таблицу, картина будет гораздо лучше и тут с большим отрывом побеждает citus (хотя в postgres скорее всего можно сократить время построения запроса, потюнив БД и создав индекс на колонку taxi_id):
1) На сервере с Postgres:
```
otus=# explain analyze select count(*), taxi_id from taxi_trips where taxi_id = 'fff84aa08ac78890c6e7da64b817cbd9aad6a124104e099a7482efee1a6bac61a837e1db218e9d38c159f28c8f85187a08a05c8ba54648edc3b91e437357fa84' group by taxi_id;
                                                                                    QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=1000.00..1558126.49 rows=3021 width=139) (actual time=209283.046..209284.830 rows=1 loops=1)
   Group Key: taxi_id
   ->  Gather  (cost=1000.00..1558078.71 rows=3514 width=139) (actual time=209283.028..209284.818 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial GroupAggregate  (cost=0.00..1556727.31 rows=1757 width=139) (actual time=209274.550..209274.552 rows=1 loops=3)
               Group Key: taxi_id
               ->  Parallel Seq Scan on taxi_trips  (cost=0.00..1556700.96 rows=1757 width=131) (actual time=7.677..209271.570 rows=1754 loops=3)
                     Filter: (taxi_id = 'fff84aa08ac78890c6e7da64b817cbd9aad6a124104e099a7482efee1a6bac61a837e1db218e9d38c159f28c8f85187a08a05c8ba54648edc3b91e437357fa84'::text)
                     Rows Removed by Filter: 8679887
 Planning Time: 0.092 ms
 Execution Time: 209284.877 ms
(12 rows)
```
2) На сервере Citus:
```
postgres=# explain analyze select count(*), taxi_id from taxi_trips where taxi_id = 'fff84aa08ac78890c6e7da64b817cbd9aad6a124104e099a7482efee1a6bac61a837e1db218e9d38c159f28c8f85187a08a05c8ba54648edc3b91e437357fa84' group by taxi_id;
                                                                                          QUERY PLAN

------------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------
 Custom Scan (Citus Router)  (cost=0.00..0.00 rows=0 width=0) (actual time=2043.516..2043.517 rows=1 loops=1)
   Task Count: 1
   Tasks Shown: All
   ->  Task
         Node: host=citus-worker-2.citus-workers port=5432 dbname=postgres
         ->  Finalize GroupAggregate  (cost=1000.00..45847.36 rows=234 width=140) (actual time=187.697..187.697 rows=1 loops=1)
               Group Key: taxi_id
               ->  Gather  (cost=1000.00..45842.68 rows=468 width=140) (actual time=187.504..187.688 rows=3 loops=1)
                     Workers Planned: 2
                     Workers Launched: 2
                     ->  Partial GroupAggregate  (cost=0.00..44795.88 rows=234 width=140) (actual time=182.390..182.390 rows=1 loops=3)
                           Group Key: taxi_id
                           ->  Parallel Seq Scan on taxi_trips_102051 taxi_trips  (cost=0.00..44783.38 rows=2032 width=132) (actual time=8.886..181.838 rows=1754 loops=
3)
                                 Filter: (taxi_id = 'fff84aa08ac78890c6e7da64b817cbd9aad6a124104e099a7482efee1a6bac61a837e1db218e9d38c159f28c8f85187a08a05c8ba54648edc3b
91e437357fa84'::text)
                                 Rows Removed by Filter: 250201
             Planning time: 0.103 ms
             Execution time: 191.502 ms
 Planning time: 0.243 ms
 Execution time: 2043.543 ms
(19 rows)
```
