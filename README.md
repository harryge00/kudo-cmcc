# kudo-cmcc

CMCC is currently using elasticsearch and influxdb. And they plan to migrate these stateful applications to k8s, using operators. Kudo can satisfy their requirements with only one universal operator. 

## Elastic Case

### Deploy an Instance
Deploy the Instance using the following command:
```
kubectl kudo install elastic --instance myes --parameter COORDINATOR_NODE_COUNT=0
```
`COORDINATOR_NODE_COUNT=0` because they only need master and data nodes.
Once the deployment has finished use the following command.
```
kubectl get pods
```
You should see that 3 master, 2 data nodes have been deployed.
```
NAME                 READY   STATUS    RESTARTS   AGE
myes-data-0          1/1     Running   0          24m
myes-data-1          1/1     Running   0          24m
myes-master-0        1/1     Running   0          25m
myes-master-1        1/1     Running   0          24m
myes-master-2        1/1     Running   0          24m
```

### Use the Instance
Exec into one of the POD's.
```
kubectl exec -ti myes-master-0 bash
```

Use the following curl command to check the health of the cluster.
```
curl myes-master-hs:9200/_cluster/health?pretty
```

You should see the following output.
```
{
  "cluster_name" : "myes-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

Lets add some data.
```
curl -X POST "myes-master-hs:9200/twitter/_doc/" -H 'Content-Type: application/json' -d'
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
'
```

Lets search for the entry.
```
curl -X GET "myes-master-hs:9200/twitter/_search?q=user:kimchy&pretty"
```

You should see the following output.
```
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "twitter",
        "_type" : "_doc",
        "_id" : "n18aemoBCj0qv5VrMWv2",
        "_score" : 0.2876821,
        "_source" : {
          "user" : "kimchy",
          "post_date" : "2009-11-15T14:12:12",
          "message" : "trying out Elasticsearch"
        }
      }
    ]
  }
}
```

## Influxdb case
### Deploy an Instance

```
kubectl kudo install ./influx
```

Once the deployment has finished use the following command.
```
kubectl get instance
```
You should see the influxdb instance:
```
NAME            AGE
influx-6j7ksk   8m32s
```

### Take a backup

Define and execute a custom plan in order to take a backup:
```
cat <<EOF | kubectl apply -f -
apiVersion: kudo.dev/v1alpha1
kind: PlanExecution
metadata:
  labels:
    operator-version: influx-0.1.0
    instance: influx-6j7ksk
  name: influx-backup
  namespace: default
spec:
  instance:
    kind: Instance
    name: influx-6j7ksk
    namespace: default
  planName: backup
EOF
```

### Perform a restore

Similar to the backup step, define and execute the restore:
```
cat <<EOF | kubectl apply -f -
apiVersion: kudo.dev/v1alpha1
kind: PlanExecution
metadata:
  labels:
    operator-version: influx-0.1.0
    instance: influx-6j7ksk
  name: influx-restore
  namespace: default
spec:
  instance:
    kind: Instance
    name: influx-6j7ksk
    namespace: default
  planName: restore
EOF
```