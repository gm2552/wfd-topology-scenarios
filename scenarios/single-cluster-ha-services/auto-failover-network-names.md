# Automatic HA Failover, Network Service Names

This scenario implements of an automatic database failover use case that uses a Kubernetes service as a single connection endpoint.  When failover is detected, the 
database controller automatically promotes a replica to be the RW instances and updates the service to point to the newly promoted replica.  The workloads
are only configured with a single connection point: the Kubernetes service name.

This scenario uses the CloudNative PG controller to spin up postgres database.

## Deploy Database And Workload

Install the CloudNative PG controller by running the following command:

```
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.1.yaml
```

Create an HA instance of postgres with 3 replicas along with a single replice Redis and RabbitMQ instance by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):

```
kubectl apply -f single-cluster-ha-services/cloudNativePG.yaml -n <namespace>
kubectl apply -f single-cluster-ha-services/rabbitMQ.yaml -n <namespace>
kubectl apply -f single-cluster-ha-services/redis.yaml -n <namespace>
```

Deploy the workloads by running the following command:

```
kubectl apply -f single-cluster-ha-services/package-install
```

## Failover Test

ClougNative PG will automatically initiate a fail over and promote a different replicate to become the RW replica when the current RW failsreplica .  When the new 
RW replica has been successfully promoted, the endpoint on `-rw` Kubernetes `Service` is updated to reference the IP address of promote replica.  The workloads
will temporarily be in a bad state during the fail over, but will automatically reconnect when the service references the promoted replica.

Execute the following steps to test a fail over scenario.

Get the IP address of the current RW instance by running the following command:

```
kubectl get pod -l cnpg.io/cluster=cnpg-where-for-dinner,role=primary --output=custom-columns='NAME:.metadata.name,IP:.status.podIP' -n <namespace>
```

Validate that the RW Kubernetes `Service Endpoint` IP matches the IP address above by running the following command:

```
kubectl get ep cnpg-where-for-dinner-rw -n <namespace>
```

To initiate a fail over, forcefully kill the RW replica by running the following command:

```
kubectl delete pod -l cnpg.io/cluster=cnpg-where-for-dinner,role=primary --force=true -n <namespace>
```

Validate that the postgres cluster is in a fail over mode by running the following command:

```
kb get cluster cnpg-where-for-dinner --watch -n <workloads>
```

The fail over sequence may take a few minutes (possibly 5 or more minutes) to complete.  You can run the command above again to check for updates and changes (or use
the `--watch` flag).  The cluster will be in a `Failing over` status during the fail over process.  When the cluster has recovered, it will be in a status of
`Cluster in healthy state`.

Get the endpoint again an validate that the IP address is now different.

```
kubectl get ep cnpg-where-for-dinner-rw -n <namespace>
```

Access the Where For Dinner application and validate that you can still retrieve searches and search results.