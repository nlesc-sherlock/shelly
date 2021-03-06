Create hadoop/spark cluster using Ambari and Docker swarm with Ansible.

# Prep

```
ssh-keygen -f shelly.key
sudo pip install ansible
```

# Machines

## For vagrant

3 nodes, 1 master + 2 slaves with 4Gb RAM each using eth0 as bridge network.

```
export SHELLY_DOMAIN=<domain>
vagrant up
```

Setup and verify login
```
ssh -i shelly.key root@shelly.<domain> uptime
ssh -i shelly.key root@shelly1.<domain> uptime
ssh -i shelly.key root@shelly2.<domain> uptime
```

Get rid of nat network

1. Login to machine
2. Remove VAGRANT section from /etc/network/interfaces
3. Stop machine
4. In Virtualbox manager disable eth0
5. Start machine

Dont use `vagrant up` anymore.

## For cloud

Machine with ssh login as root with shelly.key file

Setup and verify login
```
ssh -i shelly.key root@<host> uptime
```

## Check vars

Trusted networks and shelly user password in `group_vars/all.yml` file.

# Provision

Ansible inventory file `hosts` contains:
```

[spark-frontend]
shelly.<domain>
[spark-backends]
shelly1.<domain>
shelly2.<domain>
[docker-manager]
emma0.<domain>
[docker-node]
emma1.<domain>
```

Sanity check
```
ansible all --private-key=shelly.key -u root -i hosts -m ping
```

Install ambari server, docker, R, etc.
```
ansible-playbook --private-key=shelly.key -i hosts spark-playbook.yml
```

# Ambari setup

1. Goto http://shelly.<domain>:8080 login with admin:admin.
2. Change admin password (optional)
3. Use `shelly` as cluster name
4. Use shelly.key as private key to register hosts
5. Register hosts in ansible inventory
6. Create cluster master/slave/clients as recommended by ambari

Keep frontend node free from datanode,namenode
Make sure to add clients to frontend node.

Increase yarn mem, CPU percent and vcores when on small VM (4Gb, 1cpu)

## Hive

Mysql root:<password>

## Grafana

Grafana admin:<password>

## HDFS config

Make sure all hdfs data nodes can lookup the ip->hostname and hostname->ip of each other.
If not add the following property to hdfs-site:
```
dfs.namenode.datanode.registration.ip-hostname-check=false
```

NFS Gateway complained without this setting:
```
dfs.access.time.precision=1
```

And restart hdfs.

## Yarn config

```
node memory=6400MB
container max mem=6500Mb
virtual cores=2
container max vcores=2
yarn.nodemanager.vmem-check-enabled=true
```
## Spark config

```
spark.yarn.driver.memoryOverhead=768
spark.yarn.executor.memoryOverhead=768
```
Jupyter notebooks use very low amount of memory. Increase default with:
```
```

Ansible puts spark assembly jar in hdfs make Spark aware of it.
```
spark.yarn.jar=hdfs:///hdp/apps/2.4.2.0-258/spark/spark-hdp-assembly.jar
```

## Multihoming

Machine with multiple networks can give problems.
Deamons will bind to wrong network network.

To fix add props from
https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsMultihoming.html
to hdfs config as custom props.
For yarn/mapred check props mentioned on http://hortonworks.com/blog/multihoming-on-hadoop-yarn-clusters/

# Post ambari setup

When ambari cluster is up and running.

```
ansible-playbook --private-key=shelly.key -i hosts playbook.post.ambari.install.yml
```

# Ambari files view

0. See http://docs.hortonworks.com/HDPDocuments/Ambari-2.1.0.0/bk_ambari_views_guide/content/_configuring_your_cluster_for_files_view.html

1. Create shelly ambari user
2. Allow shelly user permission on files view.
2.1. Goto http://shelly1.sherlock-nlesc.vm.surfsara.nl:8080/views/ADMIN_VIEW/2.2.2.0/INSTANCE/#/clusters/shelly/manageAccess add shelly as read-only
2.2. Goto http://shelly1.sherlock-nlesc.vm.surfsara.nl:8080/views/ADMIN_VIEW/2.2.2.0/INSTANCE/#/views/FILES/versions/1.0.0/instances/AUTO_FILES_INSTANCE/edit grant shelly with Use permission


# Dplyr spark

spark backend for dplyr, https://github.com/RevolutionAnalytics/dplyr-spark

```
. /usr/hdp/current/hive-client/conf/hive-env.sh
sparkR --master yarn-client --driver-memory 512m --executor-memory 512m
library(dplyr)
library(dplyr.spark)
my_db = src_SparkSQL()
Error in .jfindClass(as.character(driverClass)[1]) : class not found
```

# Docker swarm

Create swarm token with:
```
SWARM_TOKEN=$(docker run --rm swarm create)
```

Provision with:
```
ansible-playbook --private-key=shelly.key -i hosts -e swarm_token=$SWARM_TOKEN docker-playbook.yml
```

The `docker-manager` manager host has swarm running on port 4000.

The docker machines will have a `/shelly/data` directory which is a nfs mount of hdfs.
The docker machines will have a `/shelly/config` directory which contains config files for hdfs/yarn/spark.
