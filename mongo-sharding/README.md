Запустить docker compose up -d

Инициализация конфигурационных серверов

```bash
docker compose exec -T configSrv1 mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id : "config_server",
  configsvr: true,
  members: [
    { _id : 0, host : "configSrv1:27017" },
    { _id : 1, host : "configSrv2:27021" },
    { _id : 2, host : "configSrv3:27022" }
  ]
});
exit();
EOF
```

Инициализация 1-го шарда

```bash
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id : "shard1",
  members: [
    { _id : 0, host : "shard1:27018" }
  ]
});
exit();
EOF
```

Инициализация 2-го шарда

```bash
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id : "shard2",
  members: [
    { _id : 1, host : "shard2:27019" }
  ]
});
exit();
EOF
```

Инициализация на роутере 1 - покажет ≥ 1000

```bash
docker compose exec -T mongos_router1 mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1:27018");
sh.addShard("shard2/shard2:27019");

sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" });

use somedb;

for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({ age: i, name: "ly" + i });

db.helloDoc.countDocuments();
exit();
EOF
```

Проверка на роутере 2 - покажет ≥ 1000

```bash
docker compose exec -T mongos_router2 mongosh --port 27023 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```


Проверка на роутере 3 - покажет ≥ 1000

```bash
docker compose exec -T mongos_router3 mongosh --port 27024 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```

Проверка на шарде 1  - покажет больше 0

```bash
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```

Проверка на шарде 2 - покажет больше 0

```bash
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```

Пересобираем и перезапускаем контейнеры - docker compose up --build -d