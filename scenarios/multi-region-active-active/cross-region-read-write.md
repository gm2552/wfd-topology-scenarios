# Cross Region Read Write (no write forward)

This sub scenario implements an Aurora Global Postgres database across both regions.  The primary read/write instance resides in 
`region-1` whilst the instance in `region-2` is a read only replica of the primary.  The read only replica does NOT support write forwarding
which requires workloads in `region-2` to make cross region connections to the primary instance.  To limit the amount of cross region transactions,
each workload will support two database endpoints: one for write operations and one for read operations.  This will require the
workloads to be aware of which connection each operation will be routed to.

From a configuration perspective, workloads in both regions will be presented with the same endpoint for read/write transaction but will
given a local endpoint for read operations.  Because only one region can hold the primary read/write instance at any given time, the read/write URL 
will contain all possible read write replicated with an option to only connect to the primary replica. This will allow the workloads to automatically
fail over to the new primary when a fail over event occurs. 

## Deploy Services And Workload

**NOTE**  It is optimal that you create the service instances in the same region where the availability targets are located.  
Make sure you are in the correct region when you perform the operations below.

### Create RDS Postgres Database

To create the primary Postgres cluster:

- Search for RDS in the AWS Web Console and click `RDS` to load the Amazon RDS dashboard. 
- Click the `Create database` button, select `Standard create`, and select `Aurora (PosgreSQL Compatible)`; do NOT select PostgreSQL option.
- Select "Dev/Test" as the template.
- Give the `DB cluster identifier` a name of your choice, and supply a master username and password in `Credential Settings`. 
- For "Instance configuration", select a size in the `Memory optimized classes` class; `db.r5.large` is appropriate. 
- Under `Availability & durability`, select `Create an Aurora Replica or Reader node in a different AZ.`
- Under `Connectivity`, chose `Yes` for `Public access`.  
- You can turn off performance insights under `Monitoring` if you wish. 
- Under `Additional configuration`, provide an `Initial database name` such as `dinner`; 
- You can disable backups as well if you wish.
- Click `Create database`.

After the primary cluster is available, go back to the RDS databases list and click on the DB identifier that has a role of `Regional cluster`.  Scroll down to the `Endpoints`
section and make note of the endpoint with the type `Writer` as you will need this later on when filling out the `rw-r2dbc-url` field in both Aurora credentials/secret files
after the secondary cluster is created.  Also make note of the endpoint with the type `Reader` as you will need this in the next section to populate the
`ro-r2dbc-url` field.

Using your editor of choice, update the following fields in the `amazonAuroraPrimary.yaml` file in 
the `scenarios/multi-region-active-active` folder of this repository with the credentials and connection information for the Postgres primary cluster.
You will update the `read-write.r2dbc-url` field after creating the secondary cluster.
You will need to base64 encode each secret/credential value before adding it to the `amazonAurora.yaml` file; an easy way to base64 values is to use an 
online tool such as https://www.base64encode.org.

- `read-write.password`
- `read-write.username`
- `read-only.password`
- `read-only.user`
- `read-only.r2dbc-url`  This field will have the following format: `r2dbc:postgresql://<reader hostname>:5432/<database name>` 

To create the secondary cluster: 
- Select the primary database cluster from the database list in the AWS console (this should be the cluster you just created).
- Click on the `Actions` drop down and then click `Add AWS region.`  
- Enter a name of your choice for the `Global database identifier` and then choose the region where your
passive/standby clusters are located.   
- For `Instance configuration`, select a size in the `Memory optimized classes` class; `db.r5.large` is appropriate. 
- Under `Availability & durability`, select `Create an Aurora Replica or Reader node in a different AZ.`  
- Under `Connectivity`, chose `Yes` for `Public access`.  
- Under `Additional configuration`, provide an `Initial database name` such as `dinner`
- You can turn off performance insights and backups as well if you wish.
- Finally, click `Add region`.

After the secondary cluster is available, go back to the RDS databases list and click on the DB identifier that has a role of `Secondary cluster`.  Scroll down 
to the `Endpoints` section and make note of the endpoint with the type `inactive` as you will need this later on when filling out the `rw-r2dbc-url` field.  
Also make note of the endpoint with the type `Reader` as you will need this in the next section to populate the `ro-r2dbc-url` field.

Using your editor of choice, update the following fields in the `amazonAuroraSecondary.yaml` file in 
the `scenarios/multi-region-active-active` folder of this repository with the credentials and connection information for the Postgres secondary cluster.

- `read-write.password`
- `read-write.username`
- `read-only.password`
- `read-only.user`
- `read-only.r2dbc-url`  This field will have the following form: `r2dbc:postgresql://<reader hostname>:5432/<database name>` 

In both the  `amazonAuroraPrimary.yaml` and  `amazonAuroraSecondary.yaml` files, update the `read-write.r2dbc-url` field with the host names of the primary `Writer` and
secondary `inacative` instances.  The URL will have the following format:

```
r2dbc:postgresql:failover://<primary writer host>:5432,<secondary inactive host>>:5432/<database name>?targetServerType=primary
```

Finally, apply the credential information to the primary and secondary clusters by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):

**TAP SM**
Run on the primary region cluster

```
kubectl apply -f multi-region-active-passive/amazonAuroraPrimary.yaml -n <namespace>
```

Run on the secondary region cluster

```
kubectl apply -f multi-region-active-passive/amazonAuroraSecondary.yaml -n <namespace>l
```

**TAP SaaS**

*TODO* Unknown how to do on TAP SaaS


#### RabbitMQ Service Creation

To create a RabbitMQ instance, perform the following steps in each region:

- Search for AmazonMQ in the AWS Web Console and click `Amazon MQ` to load the Amazon ElastiCache dashboard. 
- Click `Brokers` on the left side navigation menu,  then click the `Create brokers` button.
- Select `RabbitMQ` as the engine type and click `Next`.
- Select `Cluster deployment` and click `next`.
- Enter a name of your choosing in the Details `Broker name` field.
- In the `RabbitMQ access section`, enter a username and password.  You will need these later on when filling out the `username` and  `password` field of the secrets.
- In the `Additional settings` section, you can disable the maintenance window if you wish.  
- Click `next` then click `Create broker`


After the both broker instances are available, go back to the `Brokers` list and click on the name of your newly created broker.  Scroll down to the `Connections`
section and find the AMQP endpoint name.  Note the endpoint as this will be used in the addresses field in the next section.  **NOTE** Each region will have a different endpoint.

Using your editor of choice, update the fields with <> placeholders in the `amazonRabbitMQRegion1.yaml` and `amazonRabbitMQRegion2.yaml` files in the 
`scenarios/multi-region-active-active` folder of this repository with the credentials and connection information for the RabbitMQ end point.
Make sure each file contains the endpoint for the correct region.
You will need to base64 encode each secret/credential value 
before adding it to each credentials file; an easy way to base64 values is to use an online tool such as https://www.base64encode.org.

Finally, apply the credential information to the cluster by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):


**TAP SM**
Run on the first region's cluster

```
kubectl apply -f multi-region-active-active/amazonRabbitMQRegion1.yaml -n <namespace>
```

Run on the second region's cluster

```
kubectl apply -f multi-region-active-active/amazonRabbitMQRegion2.yaml -n <namespace>
```

**TAP SaaS**

*TODO* Unknown how to do on TAP SaaS


#### Redis Service Creation

**TODO:** Need an active/active cross region solution for Redis.

AWS does not currently support an active/active option for Redis even if connecting to the primary instance across regions (Redis instances only allow connections from 
within the same VPC).  For now, the Redis instance for the active/active scenario will use separate instances as this will have minimal impact on the Where For Dinner
application.

Create a single Redis instance by running the following command from the root of the `scenarios` directory substituting the <namepspace> placeholder 
with your run namespace (if applicable); if running in a non-Spaces environment, run the command against clusters in both regions.

```
kubectl apply -f multi-region-active-active/rabbitMQ.yaml -n <namespace>
```

### Deploy Workloads

Deploy the workloads by running the following command; if running in a non-Spaces environment, run the command against clusters in both regions.

```
kubectl apply -f multi-region-active-active/package-install
```

## Failover Test

Failover testing consists of promoting Postgres instances in the secondary cluster to become the primary instances.  The general flow of this test consists
of the following:

- Failover Postgres
- Test the WFD application.

### Failover Postgres

To failover to the postgres instance, execute the following steps.

- Search for RDS in the AWS Web Console and click `RDS`. 
- Load the list of databases by clicking `Databases` from the navigation menu on the left side of the screen.
- Select the radio button next to the database/cluster that you want to failover.
- Click on the `Actions` dropdown menu and the click `Switch over or fail over global database`
- Choose `Switchover` and select the secondary cluster in the `New primary cluster` drop down.
- Confirm the faileover and Click `Confirm`

After a few minutes (possibly 5 or more depending on amount of data in the database and load), the secondary cluster will become the primary cluster.  
You can validate this by viewing the list of databases and ensuring that the previous secondary cluster now has a role of `Primary Cluster`.

### Test Application

**TODO:** Update testing to validate connectivity of the `search` and `availability` workloads in both regions.

To test that the Where For Dinner application has been failed over appropriately, execute the following steps: 

- Load the Where For Dinner application instance in your browser and validate that the application loads.
- Create a new search in the seach `Submit Search Information` form and click `Submit Seacch`
- Validate that the new search appears under the list of `Submitted Searches And Results`