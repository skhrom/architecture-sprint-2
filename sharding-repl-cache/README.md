# pymongo-api

## Как запустить
Проверить что находимся в директории sharding-repl-cache

Запускаем mongodb и приложение

```shell
docker compose up -d
```

Инициализируем сервер конфигурации
```shell
docker exec -it configSrv mongosh --port 27017

> rs.initiate(
  {
    _id : "config_server",
       configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
);
> exit();
```

Инициализируем шарды c репликами
```shell
docker exec -it shard1 mongosh --port 27018

> rs.initiate({_id: "shard1", members: [
{_id: 0, host: "shard1:27018"},
{_id: 1, host: "shard1_repl1:27010"},
{_id: 2, host: "shard1_repl2:27011"},
{_id: 3, host: "shard1_repl3:27012"}
]});
> exit();

docker exec -it shard2 mongosh --port 27019

> rs.initiate({_id: "shard2", members: [
	{_id: 0, host: "shard2:27019"},
	{_id: 1, host: "shard2_repl1:27005"},
	{_id: 2, host: "shard2_repl2:27006"},
	{_id: 3, host: "shard2_repl3:27007"}
  ]});
> exit(); 
```

Инцициализируем роутер и наполняем его тестовыми данными
```shell
docker exec -it mongos_router mongosh --port 27020

> sh.addShard( "shard1/shard1:27018");
> sh.addShard( "shard2/shard2:27019");

> sh.enableSharding("somedb");
> sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )

> use somedb

> for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

> exit();
```

## Как проверить

### Через консоль

Общее количество документов

```shell
docker exec -it mongos_router mongosh --port 27020
 > use somedb;
 > db.helloDoc.countDocuments();
 > exit
```

Количество документов на каждом шарде
```shell
docker exec -it shard1 mongosh --port 27018
 > use somedb;
 > db.helloDoc.countDocuments();
 > exit;

docker exec -it shard2 mongosh --port 27019
 > use somedb;
 > db.helloDoc.countDocuments();
 > exit(); 

```
Количество документов на реплике.
Ниже приведен пример для реплики shard2_repl2, аналогично можно проверить для каждой из реплик
```shell
docker exec -it shard2_repl2 mongosh --port 27006
 > use somedb;
 > db.helloDoc.countDocuments();
 > exit;

```

Проверить статус реплики
Ниже приведен пример для реплики shard2_repl2, аналогично можно проверить для каждой из реплик
```shell
docker exec -it shard2_repl2 mongosh --port 27006
 >  rs.isMaster();
 > exit;

```

### Через приложение pymongo_api
открыть в браузере http://localhost:8080


### Проверить работу кэша
Повторные запросы должны выполняться быстрее первого
http://localhost:8080/helloDoc/users