# MySQL with automatic failover on Kubernetes

This is a galera cluster setup, with plain manifests.
We actually use it in production, though with modest loads.

## Get started

First create a storage class `mysql-data`. See exampels in `./configure/`.
You might also want to edit the volume size request, at the bottom of `./50mariadb.yml`.

Then: `kubectl apply -f .`.

### Cluster Health

Readiness and liveness probes will only assert client-level health of individual pods.
Watch logs for "sst" or "Quorum results", or run this quick check:
```
for i in 0 1 2; do kubectl -n mysql exec mariadb-$i -- mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';" -N; done
```

Port 9104 exposes plaintext metris in [Prometheus](https://prometheus.io/docs/concepts/data_model/) scrape format.
```
# with kubectl -n mysql port-forward mariadb-0 9104:9104
$ curl -s http://localhost:9104/metrics | grep ^mysql_global_status_wsrep_cluster_size
mysql_global_status_wsrep_cluster_size 3
```

A reasonable alert is on `mysql_global_status_wsrep_cluster_size` staying below the desired number of replicas.

### Cluster un-health

We need to assume a couple of things here. First and foremost:
Production clusters are configured so that the statefulset pods do not go down together.

 * Pods are properly spread across nodes - which this repo can try to test.
 * Nodes are spread across multiple availability zones.

Let's also assume there is monitoring.
Any `wsrep_cluster_size` issue (see above), or absence of `wsrep_cluster_size`
should lead to a human being paged.

Rarity combined with manual attention means that this statefulset can/should avoid
attempts at automatic [recovery](http://galeracluster.com/documentation-webpages/pcrecovery.html).
The reason for that is that we can't test for failure modes properly,
as they depend on the Kubernetes setup.
Automatic attempts may appoint the wrong leader, losing writes,
or cause split-brain situations.

We can however support detection in the init script.

Scaling down to two instances
-- actually one instance, but nodes should be considered ephemeral so don't do that --
and up to any number is considered normal operations.

### phpMyAdmin

Carefully consider the security implications before you create this. Note that it uses a non-official image.

```
kubectl apply -f myadmin/
```

PhpMyAdmin has a login page where you need a mysql user. To allow login (with full access) create a user with your choice of password:

```
kubectl -n mysql exec mariadb-0 -- mysql -e "CREATE USER 'phpmyadmin'@'%' IDENTIFIED BY 'my-admin-pw'; GRANT ALL ON *.* TO 'phpmyadmin'@'%';"
```

## Recover

It's in scope for this repo to support scaling down to 0 pods, then scale upp again (for example at maintenance or major upgrades). It's outside the scope of this repo to recover from crashes.

In the unlikely event that all pods crash, the statefulset tries to restart the pods but can fail with an infinite loop.

In order to restart the cluster, the "Safe-To-Bootstrap" flag must be set. Please follow the instructions found [here](http://galeracluster.com/2016/11/introducing-the-safe-to-bootstrap-feature-in-galera-cluster/)

Note that their use of the word _Node_ refers to a Pod and its volume in the Kubernetes world. So to change the `safe_to_bootstrap` flag, start a new pod with the correct volume attached.

### Summary (If link becomes broken)

Recreate 50mariadb.yml with `--wsrep-recover` command arg flag set

Get the log using `kubectl -n mysql logs -f mariadb-0` and look for the following output.

```
...
2016-11-18 01:42:15 36311 [Note] InnoDB: Database was not shutdown normally!
2016-11-18 01:42:15 36311 [Note] InnoDB: Starting crash recovery.
...
2016-11-18 01:42:16 36311 [Note] WSREP: Recovered position: 37bb872a-ad73-11e6-819f-f3b71d9c5ada:345628
...
2016-11-18 01:42:17 36311 [Note] /home/philips/git/mysql-wsrep-bugs-5.6/sql/mysqld: Shutdown complete
```

The number after the UUID string on the "Recovered position" line is the one to watch.

Pick the volume that has the highest such number and edit its `grastate.dat` to set `safe_to_bootstrap: 1`:
```
# GALERA saved state
version: 2.1
uuid:    37bb872a-ad73-11e6-819f-f3b71d9c5ada
seqno:   -1
safe_to_bootstrap: 1
```

Remove the `--wsrep-recover` and continue as *Initialize volumes and cluster*
