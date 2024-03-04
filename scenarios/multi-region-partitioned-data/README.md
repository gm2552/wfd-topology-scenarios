# Multi Region, Active/Active, Partitioned Data


This scenario consists of multiple run clusters spanning across two regions with services running in various HA configurations.  Both regions host
a fully functioning instance of the Where For Dinner application, however each region keeps a completely separate and discrete instance of the data. 
Both regions are always active, and the GSLB or equivalent construct contains rules to always route traffic to a specific region generally based traffic source.
Different sources of traffic may be routed to a different regions based on the routing rules.
Each region could be considered it's own instance of the Where For Dinner application as no data is replicated between regions.  This configuration implies that 
each region will have it's own set of data service.  Each discrete region is configured similarly to the 
(multi-region, HA services scenario)[../single-cluster-ha-services/README.md].

For the sake of being more inline with a real world deployment, this scenario uses AWS services.

**NOTE:** GSLB or an equivalent construct is a key point of configuration in this scenario; it is currently unknown for `spaces` deployments how the GSLB
will be configured to control traffic to a given set of availability targets.   

## Prerequisites

- Multiple kubernetes clusters in different regions or a Tanzu `Space` with availability targets spanning multiple regions.  All cluster have TAP packages installed.
  - If targeting a Space, appropriate profiles and traits have been deployed to the space.
  - If targeting a Kubernetes cluster directly, the Spring Cloud Gateway package should be installed.
- Valid kubernetes contexts have been configured to point the desired `clusters` or `space`. 
- GSLB or an equivalent construct is configured to route all traffic to availability targets within a single region based on the source of the traffic.
- An AWS account with the ability to create RDS, Elaticache, and AmazonMQ service instances.

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

### Create Service Instances

**NOTE:**  It is optimal that you create the service instances in the same region where the availability targets are located.  
Make sure you are in the correct region when you perform the operations below.

#### Create RDS Postgres Database

To create a Postgres instance, perform the following steps in each region:

- Search for RDS in the AWS Web Console and click `RDS` to load the Amazon RDS dashboard. 
- Click the `Create database` button, select `Standard create`, and select `PostgreSQL`; do NOT select the Aurora PostgreSQL option.
- Select "Dev/Test" as the template, and select `Multi-AZ DB instance` the option.  
- Give the `DB instance identifier` a name of your choice, and supply a master username and password in "Credential Settings". 
- For "Instance configuration", select a smaller size in the `Burstable classes` class; `db.t4g.large` is appropriate. 
- Under `Connectivity`, chose `Yes` for `Public access`.  You may need to update the selected security group to allow inbound traffic on port 5432.
- You can turn off performance insights if you wish. 
- Under `Additional configuration`, provide an `Initial database name` such as `dinner`; 
- You can disable backups if you wish. 
- Click `Create database`.

After the databases have been created and are available, click on the database instance from the list of databases and scroll down to the `Connectivity & security` section. 
Note the endpoint as this will be used in the host field in the next section.  **NOTE** Each region will have a different endpoint.

Using your editor of choice, update the fields with <> placeholders in the `amazonAuroraRegion1.yaml` and `amazonAuroraRegion2.yaml` files in the 
`scenarios/multi-region-partitioned-data folder of this repository with the credentials and connection information for the Postgres end point.
Make sure each file contains the endpoint for the correct region.
You will need to base64 encode each secret/credential value 
before adding it to each credentials file; an easy way to base64 values is to use an online tool such as https://www.base64encode.org.

Finally, apply the credential information to the cluster by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):

**TAP SM**
Run on the first region's cluster

```
kubectl apply -f multi-region-partitioned-data/amazonAuroraRegion1.yaml -n <namespace>
```

Run on the second region's cluster

```
kubectl apply -f multi-region-partitioned-data/amazonAuroraRegion2.yaml -n <namespace>
```

**TAP SaaS**

*TODO* Unknown how to do on TAP SaaS


#### Redis Service Creation

To create a Redis instance, perform the following steps in each region:

- Search for ElastiCache in the AWS Web Console and click `ElastiCache` to load the Amazon ElastiCache dashboard. 
- Click `Redis caches` on the left side navigation menu,  then click the `Create Redis cache` button.
- Select `Design your own cache`  and `Cluster cache`.
- In the `Cluster mode section` select `Disabled`
- Enter a name of your choosing in the Cluster Info `name` field.
- In the `Cluster settings` section, select a smaller node type: `cache.r7g.large` is an appropriate setting.
- In the `Connectivity section` select an existing subnet group or create a new one if one does not already exist and give it a name of your choosing.  
- Ensure that the VPC is the same VPC where your clusters reside.  You may need to create a new subnet group if the desired VPC is not listed.  Then click `next.`
- In the `security` section, select enabled for both `Encyrption at rest` and `Encrption in transit`.  
- For `access control` choose `Redis Auth default user access` and entering an appropriate password; you will need this password later on when filling out the `password` field of the secrets.
- In the `Selected security groups`, select a an existing security group; you may need to later update the inbound traffic rules to allow inbound traffic on port 6379.
- You can disable backups if you wish.  
- Click `next` then click `create`.


After the both cache instances are available, go back to the `Redis caches` list and click on the name of your newly created Redis cache.  Look in the `Cluster datails`
section and find the `Primary endpoint` name.  Note the endpoint as this will be used in the host field in the next section.  **NOTE** Each region will have a different endpoint.

Using your editor of choice, update the fields with <> placeholders in the `amazonRedisRegion1.yaml` and `amazonRedisRegion2.yaml` files in the 
`scenarios/multi-region-partitioned-data` folder of this repository with the credentials and connection information for the Redis end point.
Make sure each file contains the endpoint for the correct region.
You will need to base64 encode each secret/credential value 
before adding it to each credentials file; an easy way to base64 values is to use an online tool such as https://www.base64encode.org.

Finally, apply the credential information to the cluster by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):


**TAP SM**
Run on the first region's cluster

```
kubectl apply -f multi-region-partitioned-data/amazonRedisRegion1.yaml -n <namespace>
```

Run on the second region's cluster

```
kubectl apply -f multi-region-partitioned-data/amazonRedisRegion2.yaml -n <namespace>
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
`scenarios/multi-region-partitioned-data` folder of this repository with the credentials and connection information for the RabbitMQ end point.
Make sure each file contains the endpoint for the correct region.
You will need to base64 encode each secret/credential value 
before adding it to each credentials file; an easy way to base64 values is to use an online tool such as https://www.base64encode.org.

Finally, apply the credential information to the cluster by running the following 
command from the root of the `scenarios` directory substituting the <namepspace> placeholder with your run namespace (if applicable):


**TAP SM**
Run on the first region's cluster

```
kubectl apply -f multi-region-partitioned-data/amazonRabbitMQRegion1.yaml -n <namespace>
```

Run on the second region's cluster

```
kubectl apply -f multi-region-partitioned-data/amazonRabbitMQRegion2.yaml -n <namespace>
```

**TAP SaaS**

*TODO* Unknown how to do on TAP SaaS


### Deploy Workloads

Deploy the workloads by running the following command; if running in a non-Spaces environment, run the command against clusters in both regions.

```
kubectl apply -f multi-region-partitioned-data/package-install
```

