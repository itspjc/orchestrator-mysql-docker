# orchestrator-mysql-ha



# Setup

GCP에서 VM3개를 생성함.

- **mysql-node1**: orchestrator, mysql-server

  - 10.178.0.7 (34.64.119.151)

- **mysql-node2**: mysql-server

  - 10.178.0.8 (34.64.153.188)

- **mysql-node3**: mysql-server

  - 10.178.0.9 (34.64.153.188)

    

## Install orchestrator (mysql-node1)

- download and build orchestrator image

```
git clone https://github.com/openark/orchestrator.git
cd orchestrator
cp docker/Dockerfile .
docker build -t orchestrator:latest .
```

- check connection to webpage of orchestrator
  - http://34.64.119.151:3000/



- run

```
docker run -d -p 3000:3000 \
	-v ${PWD}/orchestrator.conf.json:/etc/orchestrator.conf.json \
	--name orchestrator \
	orchestrator:latest
```



## setup mysql server (mysql-node1~3)

When VM's hostname  is mysql-node`X` , set environment variable as `NODE=X`

```
NODE=X # 1~3
```

Run docker container of mysql-server

```
docker run -d --name=mysql$NODE \
	--net=host \
	--hostname=mysql-node$NODE \
	-p 3306:3306 \
	-v $PWD/d$NODE:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
	mysql/mysql-server:8.0 \
	--server-id=$NODE \
	--enforce-gtid-consistency='ON' \
	--log-slave-updates='ON' \
	--gtid-mode='ON' \
	--log-bin='mysql-bin-1.log'

```



## mysql-node1 (master node)

The MySQL containers must be with status **(healthy)** and **NOT** ***(health: starting)*** to go the next step.

Setting master replication in **node1**:

```
docker exec -it mysql1 mysql -uroot -pmypass \
	-e "CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'slavepass';" \
	-e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
	-e "SHOW MASTER STATUS;"
```



## mysql-node2~3 (slave nodes)

When master host is `mysql-node1`, this command is valid. If you have different master host, please change MASTER_HOST. Note that It is the same command after failover recovery.



```
docker exec -it mysql$NODE mysql -uroot -pmypass \
	-e "CHANGE MASTER TO MASTER_HOST='mysql-node1', MASTER_PORT=3306, \
        MASTER_USER='repl', MASTER_PASSWORD='slavepass', MASTER_AUTO_POSITION = 1;"
```

```
docker exec -it mysql$NODE mysql -uroot -pmypass -e "START SLAVE;"
```



- Checking whether slaves are replicating. ***Slave_IO_Running: Yes*** and ***Slave_SQL_Running: Yes***. If Slave_IO_Running stucks in Connecting status, try restarting mysql-server container.

```
docker exec -it mysql2 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"

docker exec -it mysql3 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"
```





## mysql-node1 (master node: Orchestrator)

Grant access to the Orchestrator so it can see the topology:

```
docker exec -it mysql1 mysql -uroot -pmypass \
	-e "CREATE USER 'orc_client_user'@'%' IDENTIFIED BY 'orc_client_password';" \
	-e "GRANT SUPER, PROCESS, REPLICATION SLAVE, RELOAD ON *.* TO 'orc_client_user'@'%';" \
	-e "GRANT SELECT ON mysql.slave_master_info TO 'orc_client_user'@'%';"
```



```
docker exec -it orchestrator ./orchestrator -c discover -i 10.178.0.7:3306 --debug
docker exec -it orchestrator ./orchestrator -c topology -i 10.178.0.7:3306
```





### Ref

https://github.com/wagnerjfr/orchestrator-mysql-replication-docker

https://velog.io/@hansung/MySQL-with-Orchestrator-Automated-recovery

https://velog.io/@hansung/MySQL-with-Orchestrator-Automated-recovery
