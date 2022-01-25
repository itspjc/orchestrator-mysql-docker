# orchestrator-mysql-ha


# Environments
- Ubuntu 20.04
- Docker engine (20.10.12)
	- [Install Docker](https://docs.docker.com/engine/install/ubuntu/)

# Setup

Prepare 3VM to setup mysql and orchestrator with docker engine. Please install docker engine on all VMs.

- **mysql-node1**: mysql-server (Master)

  - 172.38.101.110

- **mysql-node2**: mysql-server (Slave)

  - 172.38.101.198

- **orchestrator-vm**: orchestrator

  - 172.37.101.203

    

## Install orchestrator (In orchestrator VM)

- download and build orchestrator image

```
git clone https://github.com/openark/orchestrator.git
cd orchestrator
cp docker/Dockerfile .
docker build -t orchestrator:latest .
```

- create custom config file
create **orchestrator.conf.json** for custom config file using my `orchestrator.conf.json`.
I modified only `HostnameResolveMethod` option because I don't need to resolve hostname. Please refer this [link](https://github.com/openark/orchestrator/blob/master/docs/configuration-discovery-resolve.md)
```
"HostnameResolveMethod": "none"
```

- run docker container
```
docker run -d -p 3000:3000 \
	-v ${PWD}/orchestrator.conf.json:/etc/orchestrator.conf.json \
	--name orchestrator \
	orchestrator:latest
```
- check connection to webpage of orchestrator. Maybe It displays message of 'No clusters found'. Good.
  - http://172.37.101.203:3000/


## setup mysql server (both mysql-node1 and mysql-node2)

When VM's hostname  is mysql-node`X` , set environment variable as `NODE=X`
```
NODE=X # 1~2
```

Run docker container of mysql-server

```
docker run -d --name=mysql-node$NODE \
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
Setting master replication in **mysql-node1**:

```
docker exec -it mysql-node1 mysql -uroot -pmypass \
	-e "CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'slavepass';" \
	-e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
	-e "SHOW MASTER STATUS;"
```

## mysql-node2 (slave nodes)

When master host is `mysql-node1`, this command is valid. If you have different master host, please change MASTER_HOST. **Note that It is the same command after failover recovery.**

```
docker exec -it mysql-node$NODE mysql -uroot -pmypass \
	-e "CHANGE MASTER TO MASTER_HOST='172.38.101.110', MASTER_PORT=3306, \
        MASTER_USER='repl', MASTER_PASSWORD='slavepass', MASTER_AUTO_POSITION = 1;"
```

Start SLAVE.
```
docker exec -it mysql-node$NODE mysql -uroot -pmypass -e "START SLAVE;"
```

(Option) If you want to stop Slave, please use this command.
```
docker exec -it mysql-node$NODE mysql -uroot -pmypass -e "STOP SLAVE;"
```


- Checking whether slaves are replicating. ***Slave_IO_Running: Yes*** and ***Slave_SQL_Running: Yes***. If Slave_IO_Running stucks in Connecting status, try restarting mysql-server container.

```
docker exec -it mysql-node2 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"
```


## Setup Orchestrator 

### On mysql-node1
Grant access to the Orchestrator so it can see the topology:

```
docker exec -it mysql-node1 mysql -uroot -pmypass \
	-e "CREATE USER 'orc_client_user'@'%' IDENTIFIED BY 'orc_client_password';" \
	-e "GRANT SUPER, PROCESS, REPLICATION SLAVE, RELOAD ON *.* TO 'orc_client_user'@'%';" \
	-e "GRANT SELECT ON mysql.slave_master_info TO 'orc_client_user'@'%';"
```

### On orchestrator-vm

Please replace ip `172.38.101.110` with your master ip (mysql-node1).
Now it's time to run the commands in Orchestrator container so it can find the topology:

To discover the topology:
```
docker exec -it orchestrator ./orchestrator -c discover -i 172.38.101.110:3306 --debug
```
To see the topology:
```
docker exec -it orchestrator ./orchestrator -c topology -i 172.38.101.110:3306
```

### Ref

https://github.com/wagnerjfr/orchestrator-mysql-replication-docker

https://velog.io/@hansung/MySQL-with-Orchestrator-Automated-recovery

https://velog.io/@hansung/MySQL-with-Orchestrator-Automated-recovery
