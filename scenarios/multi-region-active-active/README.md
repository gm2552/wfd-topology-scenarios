# Multi Region, Active/Active, Shared Data


This scenario consists of multiple run clusters spanning across two regions with services running in various HA configurations.  Both regions host
active replicas of all of the Where For Dinner workloads, and data is shared across both regions.  Incoming traffic can be routed to either region
by the GSLB or equivalent construct contains.  Together, the two regions make up a single logical instance of the Where For Dinner application. 

This scenario breaks down into sub scenarios to demonstrate different service and workload configuration options.

## Prerequisites

- Multiple kubernetes clusters in different regions or a Tanzu `Space` with availability targets spanning multiple regions.  All cluster have TAP packages installed.
  - If targeting a Space, appropriate profiles and traits have been deployed to the space.
  - If targeting a Kubernetes cluster directly, the Spring Cloud Gateway package should be installed.
- Valid kubernetes contexts have been configured to point the desired `clusters` or `space`. 
- GSLB or an equivalent construct is configured to route traffic to either availability target. 
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

### Sub Scenarios

Each one of the sub scenarios should be considered a unique deployment of the where for dinner application.


- [Cross Region Read/Write](cross-region-read-write.md)
- [Write Forward](cross-region-write-forward.md)
