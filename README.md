ProxySQL Admin
==============

The ProxySQL Admin (proxysql-admin) solution configures Percona XtraDB cluster nodes into ProxySQL.

proxysql-admin usage info

```bash
Usage: [ options ]
Options:
  --config-file                      Read login credentials form a configuration file (overrides any login credentials specified on the command line)
  --test-run                         This option is used to setup dummy proxysql configuration
  --proxysql-username=user_name      Username for connecting to the ProxySQL service
  --proxysql-password[=password]     Password for connecting to the ProxySQL service
  --proxysql-port=port_num           Port Nr. for connecting to the ProxySQL service
  --proxysql-hostname=host_name      Hostname for connecting to the ProxySQL service
  --cluster-username=user_name       Username for connecting to the Percona XtraDB Cluster node
  --cluster-password[=password]      Password for connecting to the Percona XtraDB Cluster node
  --cluster-port=port_num            Port Nr. for connecting to the Percona XtraDB Cluster node
  --cluster-hostname=host_name       Hostname for connecting to the Percona XtraDB Cluster node
  --cluster-app-username=user_name   Application username for connecting to the Percona XtraDB Cluster node
  --cluster-app-password[=password]  Application password for connecting to the Percona XtraDB Cluster node
  --monitor-username=user_name       Username for monitoring Percona XtraDB Cluster nodes through ProxySQL
  --monitor-password[=password]      Password for monitoring Percona XtraDB Cluster nodes through ProxySQL
  --enable, -e                       Auto-configure Percona XtraDB Cluster nodes into ProxySQL
  --disable, -d                      Remove any Percona XtraDB Cluster configurations from ProxySQL
  --node-check-interval              Interval for monitoring node checker script (in milliseconds)
  --mode                             ProxySQL read/write configuration mode, currently supporting: 'loadbal' and 'singlewrite' (the default) modes
  --write-node                       Writer node to accept write statments. This option is supported only when using --mode=singlewrite
  --adduser                          Adds the Percona XtraDB Cluster application user to the ProxySQL database
  --version, -v                      Print version info
```
Pre-requisites 
--------------
* ProxySQL and Cluster should be up and running.
* For security purposes, please ensure to change the default user settings in the ProxySQL configuration file.

It is recommend you use _--config-file_ to run this proxysql-admin script.

This script will accept two different options to configure Cluster nodes

  __1) --enable__

  This option will configure Cluster nodes into the ProxySQL database, and add two cluster monitoring scripts into the ProxySQL scheduler table for checking the cluster status.
  _scheduler script info :
  * proxysql_node_monitor : will check cluster node membership, and re-configure ProxySQL if cluster membership changes occur
  * proxysql_galera_checker : will check desynced nodes, and temporarily deactivate them

  It will also add two new users into Cluster with USAGE privilege. One is for monitoring cluster nodes through ProxySQL, and another is for connecting to Cluster node via the ProxySQL console.

  Note: Please make sure to use super user credentials from Cluster to setup the default users.

```bash  
$ sudo proxysql-admin --config-file=/etc/proxysql-admin.cnf --enable

This script will assist with configuring ProxySQL (currently only Percona XtraDB cluster in combination with ProxySQL is supported)

ProxySQL read/write configuration mode is singlewrite


Configuring ProxySQL monitoring user..
ProxySQL monitor username as per command line/config-file is monitor


User 'monitor'@'127.%' has been added with USAGE privilege


Configuring the Percona XtraDB Cluster application user to connect through ProxySQL
Percona XtraDB Cluster application username as per command line/config-file is proxysql_user


Percona XtraDB Cluster application user 'proxysql_user'@'127.%' has been added with the USAGE privilege, please make sure to the grant appropriate privileges


Adding the Percona XtraDB Cluster server nodes to ProxySQL
You have not given the writer node info through the command line or in the config-file. Please enter the writer-node info (eg : 127.0.0.1:3306): 127.0.0.1:25000

ProxySQL configuration completed!

ProxySQL has been successfully configured to use with Percona XtraDB Cluster

You can use the following login credentials to connect your application through ProxySQL

mysql --user=proxysql_user --password=xxxxx  --host=127.0.0.1 --port=6033 --protocol=tcp 

$ 

mysql> select hostgroup_id,hostname,port,status,comment from mysql_servers;
+--------------+-----------+-------+--------+---------+
| hostgroup_id | hostname  | port  | status | comment |
+--------------+-----------+-------+--------+---------+
| 11           | 127.0.0.1 | 25400 | ONLINE | READ    |
| 10           | 127.0.0.1 | 25000 | ONLINE | WRITE   |
| 11           | 127.0.0.1 | 25100 | ONLINE | READ    |
| 11           | 127.0.0.1 | 25200 | ONLINE | READ    |
| 11           | 127.0.0.1 | 25300 | ONLINE | READ    |
+--------------+-----------+-------+--------+---------+
5 rows in set (0.00 sec)

mysql> 
```
  __2) --disable__ 
  
  This option will remove Percona XtraDB Cluster nodes from ProxySQL and stop the ProxySQL monitoring daemon.
```bash
$ proxysql-admin --config-file=/etc/proxysql-admin.cnf --disable
ProxySQL configuration removed!
$ 

```

___Extra options___

__i) --mode__

This option allows you to setup read/write mode for cluster nodes in ProxySQL database based on the hostgroup. For now, the only supported modes are _loadbal_  and _singlewrite_.  _singlewrite_ is the default mode, and it will accept writes only one single node (based on the info you provide in --write-node). Remaining nodes will accept read statements. The mode _loadbal_ on the other hand is a load balanced set of evenly weighted read/write nodes.

_singlewrite_ mode setup:
```bash
$ sudo grep "MODE" /etc/proxysql-admin.cnf
export MODE="singlewrite"
$ 
$ sudo proxysql-admin --config-file=/etc/proxysql-admin.cnf --write-node=127.0.0.1:25000 --enable
ProxySQL read/write configuration mode is singlewrite
[..]
ProxySQL configuration completed!
$

mysql> select hostgroup_id,hostname,port,status,comment from mysql_servers;
+--------------+-----------+-------+--------+---------+
| hostgroup_id | hostname  | port  | status | comment |
+--------------+-----------+-------+--------+---------+
| 11           | 127.0.0.1 | 25400 | ONLINE | READ    |
| 10           | 127.0.0.1 | 25000 | ONLINE | WRITE   |
| 11           | 127.0.0.1 | 25100 | ONLINE | READ    |
| 11           | 127.0.0.1 | 25200 | ONLINE | READ    |
| 11           | 127.0.0.1 | 25300 | ONLINE | READ    |
+--------------+-----------+-------+--------+---------+
5 rows in set (0.00 sec)

mysql> 
```

_loadbal_ mode setup:
```bash
$ sudo proxysql-admin --config-file=/etc/proxysql-admin.cnf --mode=loadbal --enable
ProxySQL read/write configuration mode is loadbal
[..]
ProxySQL configuration completed!
$ 

mysql> select hostgroup_id,hostname,port,status,comment from mysql_servers;
+--------------+-----------+-------+--------+-----------+
| hostgroup_id | hostname  | port  | status | comment   |
+--------------+-----------+-------+--------+-----------+
| 10           | 127.0.0.1 | 25400 | ONLINE | READWRITE |
| 10           | 127.0.0.1 | 25000 | ONLINE | READWRITE |
| 10           | 127.0.0.1 | 25100 | ONLINE | READWRITE |
| 10           | 127.0.0.1 | 25200 | ONLINE | READWRITE |
| 10           | 127.0.0.1 | 25300 | ONLINE | READWRITE |
+--------------+-----------+-------+--------+-----------+
5 rows in set (0.01 sec)

mysql> 
```

__ii) --galera-check-interval__

This option configures the interval for monitoring via the proxysql_galera_checker script (in milliseconds)

```bash
$ proxysql-admin --config-file=/etc/proxysql-admin.cnf --galera-check-interval=5000 --enable
```
__iii) --adduser__

This option will aid with adding the Cluster application user to ProxySQL database for you

```bash
$ proxysql-admin --config-file=/etc/proxysql-admin.cnf --adduser

Adding Percona XtraDB Cluster application user to ProxySQL database
Enter Percona XtraDB Cluster application user name: root   
Enter Percona XtraDB Cluster application user password: 
Added Percona XtraDB Cluster application user to ProxySQL database!
$ 
```
__iii) --test-run__

This option is used to setup dummy proxysql configuration.

```bash
$ sudo  ./proxysql-admin  --enable --test-run

This script will assist with configuring ProxySQL (currently only Percona XtraDB cluster in combination with ProxySQL is supported)

You have selected the dry test run mode. WARNING: This will create a test user (with all privileges) in the Percona XtraDB Cluster & ProxySQL installations.

You may want to delete this user after you complete your testing!

Would you like to proceed with '--test-run' [y/n] ? y

Setting up proxysql test configuration!

Do you want to use the default ProxySQL credentials (admin:admin:6032:127.0.0.1) [y/n] ? y
Do you want to use the default Percona XtraDB Cluster credentials (root::3306:127.0.0.1) [y/n] ? n

Enter the Percona XtraDB Cluster username (super user): root
Enter the Percona XtraDB Cluster user password: 
Enter the Percona XtraDB Cluster port: 25000
Enter the Percona XtraDB Cluster hostname: localhost

ProxySQL read/write configuration mode is singlewrite

Configuring ProxySQL monitoring user..
Enter ProxySQL monitor username : monitor
Enter ProxySQL monitor user password: 

User 'monitor'@'127.%' has been added with USAGE privilege

Configuring the Percona XtraDB Cluster application user to connect through ProxySQL
Enter Percona XtraDB Cluster application username : pxc_user_test
Enter Percona XtraDB Cluster application user password: 

Percona XtraDB Cluster application user 'pxc_user_test'@'127.%' has been added with ALL privileges, this user is created for testing purposes

Adding the Percona XtraDB Cluster server nodes to ProxySQL
You have not given the writer node info through the command line or in the config-file. Please enter the writer-node info (eg : 127.0.0.1:3306): 127.0.0.1:25000

ProxySQL configuration completed!

ProxySQL has been successfully configured to use with Percona XtraDB Cluster

You can use the following login credentials to connect your application through ProxySQL

mysql --user=pxc_user_test --password=55@#@#DSW  --host=127.0.0.1 --port=6033 --protocol=tcp 

$ 
```
