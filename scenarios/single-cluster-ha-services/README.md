# Single Cluster, HA Services Topology Scenario


This scenario consists of a single run cluster with services running in various HA configurations.  It breaks down into 
sub scenarios to demostrate different HA configuration and failover options.

## Prerequisites

- A deployed kubernetes cluster or a Tanzu `Space` with TAP packages installed.
  - If targeting a Space, appropriate profiles and traits have been deployed to the space.
  - If targeting a Kubernetes cluster directly, the Spring Cloud Gateway package should be installed.
- A valid kubernetes context has been configured to point the desired `cluster` or `space`. 

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

### HA Sub Scenarios

Each one of the sub scenarios should be considered a unique deployment of the where for dinner application.


- [Automatic HA Failover, Network Service Names](auto-failover-network-names.md)
- [Automatic HA Failover, Service Proxy](auto-failover-service-proxy.md)

