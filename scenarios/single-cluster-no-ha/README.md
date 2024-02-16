# Single Cluster, No HA Topology Scenario

This scenario consists of a single run cluster with services running locally on the cluster using bitnami services.

## Prerequisites

- A deployed kubernetes cluster or a Tanzu `Space` with TAP packages installed.
  - If targeting a Space, appropriate profiles and traits have been deployed to the space.
  - If targeting a Kubernetes cluster directly, the Spring Cloud Gateway package should be installed.
- A valid kubernetes context has been configured to point the desired `cluster` or `space`. 

## Deployment

To deploy this scenario, execute the following commands from the root of the `scenarios` directory substituting the <namepspace> placeholder with your 
run namespace (if applicable).


If you are targeting a Space, use the following commands:

```
kubectl apply -f packages/
kubectl apply -f common-config/k8sGatewayRoutes.yaml
kubectl apply -f common-config/scgRoutes.yaml
kubectl apply -f single-cluster/services.yaml
kubectl apply -f single-cluster/package-install/
```

If you are targeting a Kubernetes cluster directory, first edit the `common-config/ingress.yaml` file and replace the text `<UPDATE ME>` with the full `<host>.<domain>` 
name of the application.  Eg: `where-for-dinner.perfect300rock.com`

use the following commands:

```
kubectl apply -f packages/ -n <namespace>
kubectl apply -f common-config/scgInstance.yaml -n <namespace>
kubectl apply -f common-config/scgRoutes.yaml -n <namespace>
kubectl apply -f common-config/ingress.yaml -n <namespace>
kubectl apply -f single-cluster-no-ha/services.yaml -n <namespace>
kubectl apply -f single-cluster-no-ha/package-install/ -n <namespace>
```