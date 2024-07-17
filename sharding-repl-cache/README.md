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

Инициализация репликасета для 1-го шарда (с тремя репликами)

```bash
docker compose exec -T shard1-replica1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id : "shard1",
  members: [
    { _id: 0, host: "shard1-replica1:27018" },
    { _id: 1, host: "shard1-replica2:27025" },
    { _id: 2, host: "shard1-replica3:27026" }
  ]
});
exit();
EOF
```

Инициализация репликасета для 2-го шарда (с тремя репликами)

```bash
docker compose exec -T shard2-replica1 mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id : "shard2",
  members: [
    { _id: 0, host: "shard2-replica1:27019" },
    { _id: 1, host: "shard2-replica2:27027" },
    { _id: 2, host: "shard2-replica3:27028" }
  ]
});
exit();
EOF
```

Инициализация на роутере 1 - покажет ≥ 1000
```bash
docker compose exec -T mongos_router1 mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1-replica1:27018");
sh.addShard("shard2/shard2-replica1:27019");

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
docker compose exec -T shard1-replica1 mongosh --port 27018 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```

Проверка на шарде 2 - покажет больше 0

```bash
docker compose exec -T shard2-replica1 mongosh --port 27019 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```

Пересобираем и перезапускаем контейнеры - docker compose up --build -d

Делаем первый запрос - овтет больше 100ms
http://localhost:8080/helloDoc/users

Делаем второй запрос - ответ меньше 100ms
http://localhost:8080/helloDoc/users