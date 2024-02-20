# Multi Region, Active/Passive


This scenario consists of multiple run clusters spanning across two regions with services running in various HA configurations.  Both regions host
a fully functioning instance of the Where For Dinner application, however only one region actively serves traffic at any given time.  Each region
keeps its own copy of stateful data, and data is replicated from the active region to the passive region in near real time.  Services will deployed
in AWS and use global/replication configuration.

**NOTE:** GSLB or an equivalent construct is a key point of configuration in this scenario; it is currently unknown for `spaces` deployments how the GSLB
will be configured to control traffic to a given set of availability targets.   

## Prerequisites

- Multiple kubernetes clusters in different regions or a Tanzu `Space` with availability targets spanning multiple regions.  All cluster have TAP packages installed.
  - If targeting a Space, appropriate profiles and traits have been deployed to the space.
  - If targeting a Kubernetes cluster directly, the Spring Cloud Gateway package should be installed.
- Valid kubernetes contexts have been configured to point the desired `clusters` or `space`. 
- GSLB or an equivalent construct is configured to route traffic to availability targets within a single region.

## Deployment

To deploy this scenario, execute the following commands from the root of the `scenarios` directory substituting the <namepspace> placeholder with your 
run namespace (if applicable).


### Deploy Common Config

If you are targeting a Space, use the following commands:

```
kubectl apply -f packages/
kubectl apply -f common-config/k8sGatewayRoutes.yaml
kubectl apply -f common-config/scgRoutes.yaml
```

If you are targeting a Kubernetes cluster directory, first edit the `common-config/ingress.yaml` file and replace the text `<UPDATE ME>` with the full `<host>.<domain>` 
name of the application.  Eg: `where-for-dinner.perfect300rock.com`

use the following commands:

```
kubectl apply -f packages/ -n <namespace>
kubectl apply -f common-config/ingress.yaml -n <namespace>
kubectl apply -f common-config/scgInstance.yaml <namespace>
kubectl apply -f common-config/scgRoutes.yaml -n <namespace>
```

### Create Replicated Postgres and Redis Instances

**NOTE**  It is optimal if you create the primary service replicas/cluster in your initial active region and the secondary replicas/cluster in your
passive region.  Make sure you are in the correct region when you perform the operations below.

#### Postgres Service Creation

To create a primary Postgres cluster:
- Search for RDS in the AWS Web Console and click `RDS` to load the Amazon RDS dashboard. 
- Click the `Create database` button, select `Standard create`, and select `Aurora (PosgreSQL Compatible)`; do NOT select PostgreSQL option.
- Select "Dev/Test" as the template.
- Give the `DB cluster identifier` a name of your choice, and supply a master username and password in `Credential Settings`. 
- For "Instance configuration", select a size in the `Memory optimized classes` class; `db.r5.large` is appropriate. 
- Under `Availability & durability`, select `Create an Aurora Replica or Reader node in a different AZ.`
- Under `Connectivity`, chose `Yes` for `Public access`.  
- You can turn off performance insights under `Monitoring` if you wish. 
- Under `Additional configuration`, provide an `Initial database name` such as `dinner`; 
- You can disable backups as well if you wish. Finally, 
- Click `Create database`.

Using your editor of choice, update the fields with <> placeholders in the `amazonAuroraPrimary.yaml` file in the `scenarios/multi-region-active-passive` folder
of this repository with the credentials and connection information for the Postgres primary cluster. You will need to base64 encode each secret/credential value 
before adding it to the `amazonAurora.yaml` file; an easy way to base64 values is to use an online tool such as https://www.base64encode.org.

After the primary cluster is available, go back to the RDS databases list and click on the DB identifier that has a role of `Regional cluster`.  Scroll down the `Endpoints`
section and make note of the endpoint with the type `writer` as this will be host field in your credentials file for the primary region.


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

After the secondary cluster is available, go back to the RDS databases and list click on the DB identifier that has a role of `Secondard Cluster`.  
- Scroll down the `Endpoints` section and make note of the endpoint with the type `writer` as this will be host field in your credentials file for the secondary region.

Using your editor of choice, update the fields with <> placeholders in the `amazonAuroraSecondary.yaml` file in the `scenarios/multi-region-active-passive` folder
of this repository with the credentials and connection information for the Postgres primary cluster. You will need to base64 encode each secret/credential value 
before adding it to the `amazonAurora.yaml` file; an easy way to base64 values is to use an online tool such as https://www.base64encode.org.

Finally, apply the credential information to the primary and secondary cluster by running the following 
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

#### Redis Service Creation

To create a Redis Global Datastore instance, 
- Search for ElastiCache in the AWS Web Console and click `ElastiCache` to load the Amazon ElastiCache dashboard. 
- Click `Global datastores` on the left side navigation menu,  then click the  `Create global datastore` button.
- Select `Disabled` to create a simple single shard node group and then enter a name of your choosing for the Global Datastore info `name` field.
- In the `Region cluster` section, select the region where your primary/active clusters are located and enter a name of your choosing in the Cluster Info
`name` field.
- In the `Cluster settings` section, select a smaller node type: `cache.r7g.large` is an appropriate setting.
- In the `Connectivity section` select an existing subnet group or create a new one if one does not already exist and give it a name of your choosing.  
- Ensure that the VPC is the same VPC where your clusters reside.  You may need to create a new subnet group if the desired VPC is not listed.  Then click `next.`
- In the `security` section, select enabled for both `Encyrption at rest` and `Encrption in transit`.  
  - For `access control` choose `Redis Auth default user access` and entering an appropriate password; you will need this password later on when filling out the `password` field of the secrets.
- In the `Selected security groups`, select a an existing security group; you may need to later update the inbound traffic rules to allow inbound traffic on port 6379.
- Click `next`
- In the `Regional cluster`, select the region where your standby/passive clusters are located.  Enter a name of your choosing in the Cluster Info
`name` field.
- In the `Connectivity section` select an existing subnet group or create a new one if one does not already exist and give it a name of your choosing.
- Select the VPC where your workloads will run.  Click `next`
- In the `security` section, select enabled for both `Encyrption at rest` and `Encrption in transit`.  
  - For `access control` choose `Redis Auth default user access` and enter an appropriate password; this should be the same password that you used in the primary region.
- In the `Selected security groups`, select a an existing security group; you may need to later update the inbound traffic rules to allow inbound traffic on port 6379.
- You can disable backups if you wish.  
- Click `next` then hit `create`.


After the secondary cluster is available, go back to the Global datastores list click on the name of your newly created datastore.  

- Click on the primary cluster and find the `Primary endpoint` field under `Cluster details`, you will need this in the `host` field of the primary redis secret.  

Click your browser's back button and then click on the secondary cluster name.  
- Find the `Primary endpoint` field under `Cluster details`; 
you will need this in the `host` field of the secondary redis secret. 

Using your editor of choice, update the fields with <> placeholders in the `amazonRedisPrimary.yaml` and `amazonRedisSecondary.yaml` files 
in the `scenarios/multi-region-active-passive` folder of this repository with the credentials and connection information for the redis primary and secondary clusters. 
You will need to base64 encode each secret/credential value 
before adding it to the `amazonAurora.yaml` file; an easy way to base64 values is to use an online tool such as https://www.base64encode.org.

Finally, apply the credential information to the primary and secondary cluster by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):

**TAP SM**
Run on the primary region cluster

```
kubectl apply -f multi-region-active-passive/amazonRedisPrimary.yaml -n <namespace>
```

Run on the secondary region cluster

```
kubectl apply -f multi-region-active-passive/amazonRedisSecondary.yaml -n <namespace>l
```

**TAP SaaS**

*TODO* Unknown how to do on TAP SaaS


#### RabbitMQ Service Creation

Create a single replica RabbitMQ instance by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):

```
kubectl apply -f multi-region-active-passive/rabbitMQ.yaml -n <namespace>
```

### Deploy Workloads

Deploy the workloads by running the following command:

```
kubectl apply -f multi-region-active-passive/package-install
```


## Failover Test

Failover testing consists of promoting the Redis and Postgres instances in the secondary cluster to become the primary instances.  The general flow of this test consists
of the following:

- Failover Postgres
- Failover Redis
- Restart Services
- Switch GSLB to point to route traffic to the passive cluster
- Test the WFD application.

*Test Steps Coming Soon*

### Failover Postgres

To failover to the postgres instance, execute the following steps.

- Search for RDS in the AWS Web Console and click `RDS`. 
- Load the list of databases by clicking `Databases` from the navigation menu on the left side of the screen.
- Select the radio button next to the database/cluster that you want to failover.
- Click on the `Actions` dropdown menu and the click `Switch over or fail over global database`
- Choose `Failover (allow data loss)` and select the secondary cluster in the `New primary cluster` drop down.
- Confirm the faileover and Click `Confirm`

After a few minutes or less, the secondary cluster will become the primary cluster.  You can validate this by viewing the list of databases and ensuring that the previous 
secondary cluster now has a role of `Primary Cluster`.

### Failover Redis

- Search for ElastiCache in the AWS Web Console and click `ElastiCache` 
- Click `Global datastores` on the left side navigation menu to load the list of global databases.
- Click on the datastore that you want to failover.
- Select the radio button next the secondary cluster, then click `Promote to primary`
- Click `Promote` to confirm.

After few minutes or less, the secondary cluster will become the primary cluster.  You can validate this by reloading the list of `Regional Redis Clusters` for your selectect data store
and ensuring that the previous secondary cluster now has a role of `Primary`.

### Restart Services

The fail over requires the standby services to be restarted.
Restart the following Where For Dinner workload by killing their pods.

- `where-for-dinner-search`
- `where-for-dinner-search-proc`
- `where-for-dinner-availability`

### Switchover GSLB

Using the appropriate GSLB tooling, switch traffic routing from the availability targets in the active region to now flow to availability targets in the passive/standby region.

**NOTE** It is currently unknown how to route traffic to a specific set of availability targets in TAP SaaS.

### Test Application

To test that the Where For Dinner application has been failed over appropriately, execute the following steps: 

- Load the Where For Dinner application instance in your browser and validate that the application loads.
- Create a new search in the seach `Submit Search Information` form and click `Submit Seacch`
- Validate that the new search appears under the list of `Submitted Searches And Results`