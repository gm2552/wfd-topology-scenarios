# Automatic HA Failover, Service Proxy

This scenario implements of an automatic database failover use case that uses an RDS Postgres Multi-AZ DB cluster along with an RDS Proxy.  Workloads are
configured with a single connection point: the RDS Proxy read/write endpoint.  When a failover is detected, RDS automatically promotes one of the read only replicas to
become the primary read/write replica.  When ready, the proxy reconnects to the new primary replica.  Other than a short amount of downtime during the promotion, the
workload is not aware that transactions are routed to the new replica behind the proxy.

This scenario uses AWS RDS  spin up postgres cluster; how will need access to an AWS account appropriate privileges to create a Postgres DB cluster.

## Deploy Services And Workload

**NOTE**  It is optimal that you create the service instances in the same region where the availability targets are located.  
Make sure you are in the correct region when you perform the operations below.

### Create RDS Postgres Database and Proxy

To create a Postgres instance
- Search for RDS in the AWS Web Console and click `RDS` to load the Amazon RDS dashboard. 
- Click the `Create database` button, select `Standard create`, and select `PostgreSQL`; do NOT select the Aurora PostgreSQL option.
- Select "Dev/Test" as the template, and select `Multi-AZ DB instance` the option.  
- Give the `DB instance identifier` a name of your choice, and supply a master username and password in "Credential Settings". 
- For `Instance configuration`, select a smaller size in the `Burstable classes` class; `db.t4g.large` is appropriate. 
- Under `Connectivity`, chose `Yes` for `Public access` and select `Create an RDS Proxy`.  Also, ensure that the selected VPC is the same VPC
as your cluster (proxies are not accessible across VCPs).  You may need to update the selected security group to allow inbound traffic on port 5432.
- You can turn off performance insights if you wish. 
- Under `Additional configuration`, provide an `Initial database name` such as `dinner`; 
- You can disable backups as well if you wish. 
- Click `Create database`.

After the database has been created and is available, click on the database instance from the list of databases and scroll down to the "Proxies" section. 
Click on the `Proxy identifier` and scroll down to the `Proxy endpoints` section.  Note the endpoint this will be used as the host field in the next section.

Using your editor of choice, update the fields with <> placeholders in the `amazonRDS.yaml` file in the `scenarios/single-cluster-ha-services` folder
of this repository with the credentials and connection information for the Postgres proxy. You will need to base64 encode each secret/credential value 
before adding it to the `amazonRDS.yaml` file; an easy way to base64 values is to use an online tool such as https://www.base64encode.org.

Finally, apply the credential information to the cluster by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):

```
kubectl apply -f single-cluster-ha-services/amazonRDS.yaml -n <namespace>
```

### Create Remaining Services

Create a single replica Redis and RabbitMQ instance by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):

```
kubectl apply -f single-cluster-ha-services/rabbitMQ.yaml -n <namespace>
kubectl apply -f single-cluster-ha-services/redis.yaml -n <namespace>
```

### Deploy Workloads

Deploy the workloads by running the following command:

```
kubectl apply -f single-cluster-ha-services/package-install
```

## Failover Test

In the event primary replica fails, RDS will automatically promote the standby replica to the primary replica and the proxy will redirect connections to the new 
primary replica.  The workloads will temporarily be in a bad state during the fail over, but will automatically reconnect when the proxy references the 
promoted replica.

Execute the following steps to force a fail over scenario.

In the RDS dashboard, select the database and then click the `Actions` button and then click `Reboot`.  On the next sceen, make sure
`Reboot With Failover` is checked and then click the `Confirm` button.


The fail over sequence may take a few minutes to complete.  When completed, access the Where For Dinner application and validate that you can 
still retrieve searches and search results.