# Домашнее задание к занятию "`Репликация и масштабирование. Часть 1`" - `Кудряшов Андрей`


---

### Задание 1

Master-master - оба сервера ведущие. изменения синхронизируются в обе стороны. возможен конфликт одновременного изменения одной и той же информации.

Master - slave - ведущий и ведомый. master - главный, принимает изменения. slave - ведомый, только читает изменения. slave можно перевести в статус master.


---

### Задание 2

```
sudo apt update
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker kudryashov
```
```
mkdir mysql-replication
cd mysql-replication
```
```
cat > master.cnf <<EOF
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-format=ROW
EOF
```
```
cat > slave.cnf <<EOF
[mysqld]
server-id=2
read-only=1
EOF
```
```
cat > master.sql <<EOF
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'slavepass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
EOF
```
```
cat > slave.sql <<EOF
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql_master',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='slavepass',
  SOURCE_SSL=0;

START REPLICA;
EOF
```
```
cat > Dockerfile.master <<EOF
FROM mysql:8.0
COPY master.cnf /etc/mysql/conf.d/my.cnf
COPY master.sql /docker-entrypoint-initdb.d/
ENV MYSQL_ROOT_PASSWORD=rootpass
CMD ["mysqld"]
EOF
```
```
cat > Dockerfile.slave <<EOF
FROM mysql:8.0
COPY slave.cnf /etc/mysql/conf.d/my.cnf
COPY slave.sql /docker-entrypoint-initdb.d/
ENV MYSQL_ROOT_PASSWORD=rootpass
CMD ["mysqld"]
EOF
```
```
ls -la
```
```
docker build -t mysql_master -f Dockerfile.master .
```
```
docker build -t mysql_slave -f Dockerfile.slave .
```
```
docker run -d --name mysql_master --net replication_net -p 3306:3306 mysql_master
```
через 30 сек:
```
docker run -d --name mysql_slave --net replication_net -p 3307:3306 mysql_slave
```

<img width="1273" height="99" alt="1" src="https://github.com/user-attachments/assets/30c7d5f4-0e38-475b-8643-075156ff8c0d" />

```
docker exec -it mysql_master mysql -uroot -prootpass -e "
  CREATE DATABASE test_rep;
  USE test_rep;
  CREATE TABLE cities (id INT PRIMARY KEY, name VARCHAR(50));
  INSERT INTO cities VALUES (1, 'Moscow');
"
```
```
docker exec -it mysql_slave mysql -uroot -prootpass -e "SELECT * FROM test_rep.cities;"
```

<img width="1077" height="261" alt="2" src="https://github.com/user-attachments/assets/6fb0aca7-3883-44ec-aeb6-61ec98b6ad22" />

```
docker exec -it mysql_slave mysql -uroot -prootpass -e "SHOW REPLICA STATUS\G" | grep "Running\|Error"
```

<img width="1210" height="183" alt="3" src="https://github.com/user-attachments/assets/e087682a-20a3-4baa-ad9a-4a42e771f832" />


---

### Задание 3

```
docker stop mysql_master mysql_slave
docker rm mysql_master mysql_slave
docker network rm replication_net
```

```
mkdir ~/mysql-master-master
cd ~/mysql-master-master
```

ID.
```
cat > master1.cnf <<EOF
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-format=ROW
auto-increment-increment=2
auto-increment-offset=1
EOF
```
```
cat > master2.cnf <<EOF
[mysqld]
server-id=2
log-bin=mysql-bin
binlog-format=ROW
auto-increment-increment=2
auto-increment-offset=2
EOF
```
```
cat > Dockerfile.master1 <<EOF
FROM mysql:8.0
COPY master1.cnf /etc/mysql/conf.d/my.cnf
ENV MYSQL_ROOT_PASSWORD=rootpass
CMD ["mysqld"]
EOF
```
```
cat > Dockerfile.master2 <<EOF
FROM mysql:8.0
COPY master2.cnf /etc/mysql/conf.d/my.cnf
ENV MYSQL_ROOT_PASSWORD=rootpass
CMD ["mysqld"]
EOF
```
Сборка.
```
docker build -t master1 -f Dockerfile.master1 .
docker build -t master2 -f Dockerfile.master2 .
```
Запуск.
```
docker network create mm_net
```
```
docker run -d --name master1 --net mm_net -p 3306:3306 master1
```
через 30 сек.
```
docker run -d --name master2 --net mm_net -p 3307:3306 master2
```

Пользователи:
```
docker exec -it master1 mysql -uroot -prootpass -e "
SET SQL_LOG_BIN=0;
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'pass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
"
```
```
docker exec -it master2 mysql -uroot -prootpass -e "
SET SQL_LOG_BIN=0;
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'pass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
"
```

Репликация:
```
docker exec -it master1 mysql -uroot -prootpass -e "
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='master2',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='pass',
  SOURCE_SSL=0;
START REPLICA;
"
```
```
docker exec -it master2 mysql -uroot -prootpass -e "
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='master1',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='pass',
  SOURCE_SSL=0;
START REPLICA;
"
```
Проверка:
```
docker exec -it master1 mysql -uroot -prootpass -e "SHOW REPLICA STATUS\G" | grep "Running\|Error"
```
```
docker exec -it master2 mysql -uroot -prootpass -e "SHOW REPLICA STATUS\G" | grep "Running\|Error"
```

Тест:
```
docker exec -i master1 mysql -uroot -prootpass <<< "
CREATE DATABASE IF NOT EXISTS testmm;
USE testmm;
CREATE TABLE t (id INT AUTO_INCREMENT PRIMARY KEY, msg VARCHAR(50));
INSERT INTO t (msg) VALUES ('From Master1');
"
```
```
docker exec -i master2 mysql -uroot -prootpass -e "SELECT * FROM testmm.t;"
```
```
docker exec -i master2 mysql -uroot -prootpass <<< "
USE testmm;
INSERT INTO t (msg) VALUES ('From Master2');
"
```
```
docker exec -i master1 mysql -uroot -prootpass -e "SELECT * FROM testmm.t;"
```

<img width="958" height="405" alt="10" src="https://github.com/user-attachments/assets/e09aa3b3-e542-4be3-ac38-e6289079dd40" />

<img width="955" height="546" alt="11" src="https://github.com/user-attachments/assets/c3ee9021-0237-4d74-b7be-50d86d75e61e" />


---
