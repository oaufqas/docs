### Сети в docker

Основные типы сетевого подключения (drivers): 

1. bridge (мост) - дефолтный драйвер сети докера
docker0: 172.17.0.0/16
docker run **-p 80:80** nginx


2. host (айпи с сервера)
ServerIp: 10.15.11.12
docker run **--network=host** nginx


3. none (без сетевых интерфейсов)
docker run **--network=none** myTestProd


4. macvlan (У каждого контейнера свой MAC адресс)
5. ipvlan (MAC - одинаковый, ip - разные)
6. overlay (docker swarm cluster)


## Создание сети:

docker **network create --drive bridge** myTestNet

Чтобы запускать контейнеры в созданной нами сети, а не default:
docker run **--net myTestNet** nginx

В созданной нами сети, можно исользовать DNS - обращаться к контейнерам по их именам

Чтобы подключить/отключить уже существующий контейнер к какой либо сети:
docker network **connect/disconnect** myTestNet myTestProd

--network-alias <name>: Задать «никнейм» контейнеру внутри этой сети для DNS.
--ip <address>: Назначить статический IP внутри сети.
--dns <address>: Указать свои DNS-серверы для контейнера.

# Показать детали сети
docker network inspect my-network
# Показать все сети
docker network ls
