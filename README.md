# wfd-topology-scenarios
Topology deployment scenarios for Where For Dinner.  It uses a common build configuration which references the main Where For Dinner
source location in the [sample accelerators repository](https://github.com/vmware-tanzu/application-accelerator-samples/tree/main/where-for-dinner).
The configuration uses deployed instances of RabbitMQ, MySQL, and Redit and uses ResourceClaims to reference the provisioned services (could be a direct
secret reference as well).

This project is separated into two sections: build and deploy.  The build generates Carvel packages that will be deployed into the run clusters of the various
deployment scenarios.  The run section contains configuration for deploying the Where For Dinner Carvel packages into various deployment topologies.

## Build

The run scenarios will default to using prebuilt images, however you can build your images using a TAP build cluster that is configured to build Carvel packages.
To build the Where For Dinner Carvel packages, run the following command from the root of this repository against your build cluster substituting the 
`<namepspace>` placeholder with your build namespace:

```
kubectl apply -f build/workloads.yaml -n <namespace>
```

# Run Scenarios

The [scenarios](scenarios/README.md) folder contains multiple sub-folders corresponding to a specific topology deployment scenario.  Each folder contains deployment
configuration specific to the scenario along with its own README instruction.  You can use the default images provided in the workload packages (recommended)
, or you can update each
workload package with the URL of your own workload image.