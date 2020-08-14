# postgreSQL-cluster
A postgreSQL DataBase Cluster (also a DataBase cluster for FreeSWITCH)
This solution has been designed to work as a **Stack** in **Docker Swarm Environment**.

Prepare Docker Swarm.
To provide HA you should use 2 or 3 servers.
Notice that Swarm is manages from an another server. However DB cluster would also work on 3 Managers (not recommended for production) 

1. Name the nodes (this must be done from a Swarm MANGER node).

a. Set up 'Placement constraints' for control and balancing.
   Those will be places of the nodes where loadbalancer/s and contoller container will be on.
   THe LoadBalancer can be 1 or more instances. The controller (etcd) should be 1 instance. 
```bash
    [root@swarmmanager1 ]# docker node update --label-add pdb=node workerhost1.localdomain
    [root@swarmmanager1 ]# docker node update --label-add pdb=node workerhost2.localdomain
    [root@swarmmanager1 ]# docker node update --label-add pdb=node workerhost3.localdomain
```


b. Set up 'Placement constraints' for database containers.
   Those will be places of the nodes where each instance of the database will be on.
```bash
    [root@swarmmanager1 ]# docker node update --label-add pdb1=node workerhost1.localdomain
    [root@swarmmanager1 ]# docker node update --label-add pdb2=node workerhost2.localdomain
    [root@swarmmanager1 ]# docker node update --label-add pdb3=node workerhost3.localdomain
```


2. Deploy docker swarm network
   Via this network your application will reach the database
```bash
    [root@swarmmanager1 ]# docker network create --scope swarm --attachable -d overlay postgresApp
```

1. Deploy Docker Stack (this must be done from a Swarm MANGER node).
```bash
    [root@swarmmanager1 ]# docker stack deploy -c docker-compose.yml LABpdb
    Creating service LABpdb_dbnode3
    Creating service LABpdb_postgrelb
    Creating service LABpdb_etcd
    Creating service LABpdb_dbnode1
    Creating service LABpdb_dbnode2
```


> [!IMPORTANT]
>1. **To this swarm network you should attach other containers** which will be using the PostgreSQL DataBase.
>All the traffic is going via LoadBalancer (haproxy).
>There is no direct access to the Database instance. Database and Replication traffic is in secure swarm network.
>To connect to PostgreSQL use your 'StackName_servicename_postgrelb' and ports below

    - Master:             5000
    - Slaves (Replicas):  5001
    - API:                8008
>2. The Stack is also configured to **publish the ports out of the Swarm**.
>Those ports will also become available on each IP and each node of you Docker Swarm Cluster. Please make sure this is secure network.
>To not publish the ports please remove this section from file docker-compose.yml -> 'postgrelb' -> 'ports' 

This fully functinal PostgreSQL cluster.
It works as 1 MASTER instance and multi SLAVEs instances.
This example uses 1 Master and 2 Slaves. However it can be scalled up to use more Slaves. Contact the Maintainer how to. 


Connect to cluster using 'psql' cmd (postgresql-client)

1. Install client (a. external host or b. container connected to Swarm network):
```bash
    $ sudo apt-get update && apt-get install -y postgresql-client
    $ psql --version
    psql (PostgreSQL) 9.5.12
```

2. Connect from a remotehost:
```bash
    # psql -h <StackName_servicename_postgrelb> -p 5000 -d <database> -U <user> -W
```

   example:
```bash
    # psql -h LABpdb_postgrelb -p 5000 -d postgres -U ppostgres -W
    Password for user ppostgres: <SUPERUSER_PASSWORD defined in './config-cluster/live.env' file>
    psql (9.6.3)
    postgres=#
```





Example of usage:
In this repository the DataBase Cluster is being used for FreeSWITCH (and Fusionpbx) as a HA replicated DataBase.
To use with a FreeSWITCH instance/s follow the further staps.
Init the FS databases:
```console
postgres=# CREATE DATABASE fusionpbx;
CREATE DATABASE
postgres=# CREATE DATABASE freeswitch;
CREATE DATABASE
postgres=# CREATE ROLE fusionpbx WITH SUPERUSER LOGIN PASSWORD 'changepass';
CREATE ROLE
postgres=# CREATE ROLE freeswitch WITH SUPERUSER LOGIN PASSWORD 'changepass';
CREATE ROLE
postgres=# GRANT ALL PRIVILEGES ON DATABASE fusionpbx to fusionpbx;
GRANT
postgres=# GRANT ALL PRIVILEGES ON DATABASE freeswitch to fusionpbx;
GRANT
postgres=# GRANT ALL PRIVILEGES ON DATABASE freeswitch to freeswitch;
GRANT
```

#add the databases, users and grant permissions to them
apt update && apt install -y postgresql-client
DROP SCHEMA public cascade;
CREATE SCHEMA public;
CREATE DATABASE fusionpbx ;
CREATE DATABASE freeswitch ;
CREATE ROLE fusionpbx WITH SUPERUSER LOGIN PASSWORD 'password';
CREATE ROLE freeswitch WITH SUPERUSER LOGIN PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE fusionpbx to fusionpbx;
GRANT ALL PRIVILEGES ON DATABASE freeswitch to fusionpbx;
GRANT ALL PRIVILEGES ON DATABASE freeswitch to freeswitch;

