# Run Scenarios

This folder provides common Carvel `package` resources in the `packages` directory that can be used across all scenarios; `packageinstall` 
resources are scenario specific and apply environment values to the workload deployments.

Sub folders contain deployment configuration for various deployment topologies.  These include:

- [Single Cluster, No HA](single-cluster-no-ha/README.md)
- [Single Cluster, HA Services](single-cluster-ha-services/README.md)
- [Multi-Region, Active/Passive](multi-region-active-passive/README.md)
- [Multi-Region, Partitioned Data](multi-region-partitioned-data/README.md)

See the READMEs in each scenario folder for specific deployment instructions.

