Ход выполнения ДЗ №2

Разворачиваем ВМ в сервисе Yandex Cloud. 
Все операции выполняем с помощью интерфейса CLI. 

Устанавливаем CLI на Linux по инструкции: https://cloud.yandex.ru/docs/cli/quickstart#install

Проверяем работу:

`yc config list`

Нам понадобиться ssh ключ, сгенерируем его командой: 

```
ssh-keygen -t ed25519
```

Для развертывания ВМ выполняем команду: 

```
yc compute instance create \
  --name mongo-instance \
  --hostname mongo-instance \
  --create-boot-disk size=15G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2204-lts \
  --network-interface subnet-name=mongo-net,nat-ip-version=ipv4 \
  --zone ru-central1-a \
  --metadata-from-file ssh-keys=/home/debian/.ssh/otus.pub
```
  
Пошло создание ВМ, получаем такой вывод: 

```
done (44s)
id: fhm5hsa23m9thbr70k3e
folder_id: b1gbd2f39mua8mv3vu1e
created_at: "2023-11-19T09:18:43Z"
name: mongo-instance
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "2147483648"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhmj0eo7m6ctbl4mujsp
  auto_delete: true
  disk_id: fhmj0eo7m6ctbl4mujsp
network_interfaces:
  - index: "0"
    mac_address: d0:0d:58:f1:42:1d
    subnet_id: e9b04k6bhfaqffo6utra
    primary_v4_address:
      address: 10.1.2.21
      one_to_one_nat:
        address: 158.160.119.112
        ip_version: IPV4
gpu_settings: {}
fqdn: mongo-instance.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```

Пробуем подключаться по SSH с помощью команды: 

```
ssh -i /home/debian/.ssh/otus yc-user@158.160.119.112
```

Получаем ошибку: 

```
yc-user@158.160.119.112: Permission denied (publickey).
```

Как ни пытался, ошибку победить не смог. Удалил ВМ и создал новую через консоль управления. Указал при создании имя пользователя "ubuntu" и тот же самый публичный ключ. 

Пробую подключаться на новый IP под пользователем ubuntu: 

```
ssh -i /home/debian/.ssh/otus ubuntu@158.160.104.179
```

С первого раза подключение прошло успешно. 

## Далее на ВМ устанавливаем MongoDB: 

1) Импортируем GPG ключ:

```
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor
```

2) Добавляем репозиторий: 

```
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

3) Обновляем пакеты: 

```
sudo apt update
```

4) Устанавливаем MongoDB из репозитория: 

```
sudo apt install mongodb-org -y
```


## Работа с MongoDB:

Запускаем службу: 

```
sudo systemctl start mongod
```

Проверяем статус: 

```
sudo systemctl status mongod
```

Все ок, служба работает!

Каталог с базой расположен по пути: /var/lib/mongodb/

Подключаемся к базе: 

```
mongosh
```

В базу test добавим коллекцию cars и добавим в нее запись:

```
use test
db.cars.insertOne({'Car':'Ford', 'Year':'2012'});
db.cars.find()
```

## Настроим внешние подключения к базе: 

Для этого создадим пользователя с админскими правами:

```
use admin

db.createUser({
  user: "root",
  pwd: "pGj36AjnfR7Wtd9K",
  roles: ["userAdminAnyDatabase", "dbAdminAnyDatabase", "readWriteAnyDatabase"]
})

exit
```

В конфигурационный файл /etc/mongod.conf добавим секцию security:

```
security:
  authorization: enabled
```

Перезапустим службу: 

```
sudo systemctl restart mongod
```


## На другом хосте:

Устанавливаем клиент MongoDB:

1) Импортируем GPG ключ:

```
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor
```

2) Добавляем репозиторий: 

```
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

3) Обновляем пакеты: 

```
sudo apt update
```

4) Устанавливаем MongoDB из репозитория: 

```
sudo apt install -y mongodb-mongosh
```

Пробуем подключиться к внешней базе: 

```
mongosh "mongodb://root:pGj36AjnfR7Wtd9K@158.160.104.179:27017/admin"
```

Получаем ошибку: 

	MongoNetworkError: connect ECONNREFUSED 158.160.104.179:27017

Проблема была в настройке /etc/mongod.conf в секции net, по-умолчанию включен доступ только с localhost:

```
net:
  port: 27017
  bindIp: 127.0.0.1
```

Меняем bindIp: 127.0.0.1 на bindIp: 0.0.0.0 и перезапускаем службу. 
Пробуем снова подключаться с другого хоста. Теперь подключение прошло удачно!

Проверяем данные в базе: 

```
use test
db.cars.find()
```

Все на месте: 

```
[
  {
    _id: ObjectId('655ce8bd41baa73f1599507a'),
    Car: 'Ford',
    Year: '2012'
  }
  
]
```

