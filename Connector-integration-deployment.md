## Steps to integrate and deploy a connector from Cloud Pak for Watson AIOPs

## Create the necessary container images and deployment artifacts
- Build the connector container image as in the cicd DevOps process
-  Publish your image to Docker registry. The container images from IBM can be in the IBM Cloud Container Registry, while container images from Business Partners and the user community can come from any registry.
- Formulate the required Kubernetes deployment artifacts for the connector container deployment.
- Every connector has an equivalent UI tile that produces configuration, which is then passed to the client by using the Configuration gRPC method. The data element in the Cloud event streamed in this method is a JSON serialization of the ConnectorConfiguration.spec that was created from a completed UI tile. Create the UI schema/form by following the ui-schema creation guidelines.
- Follow the bundleManifest guidelines [here](https://github.com/IBM/cp4waiops-connectors/bundle-manifest) to create bundleManifest deployment artifacts.



## Publish the deployment artifacts
- All new connectors in Cloud Pak for Watson AIOps (3.3+) are based on the new gRPC framework and are installed by using a bundleManifest.
 - Publish the bundleManifest deployment artifacts by creating the pull request to the **public** GitHub repository located [here](https://github.com/IBM/cp4waiops-connectors).
- All artifacts (BundleManifest, deployment files referencing container images, and so on) all perfectly fit a GitOps approach, thus our pipeline can hook right into this repository.
- Each quarterly release of the Cloud Pak for Watson AIOps can pick up relevant contributions from this GitHub repository, provided all the requirements are met.

## Deployment options and bootstrap

Two ways are avilable to deploy a gRPC connector or automation action:

### Local-edge:
- In this deployment mode, the container is deployed in the same namespace as Cloud Pak for Watson AIOps, so it is necessary that a set of Kubernetes deployment artifacts (by using Kustomize) is made available in the bundleManifest (more details in this section). The bootstrap information is injected to the container by using Kustomize patches to the deployment artifact.
- Refer o the example `java-grpc-connector-template` deployment artifact `/bundle-artifacts/connector/deployment.yaml` for the required files to be volume mounted

### Micro-edge:
- In this deployment mode, the container is started manually by the user, most of the time in a remote VM that sits in a different network boundary. This mode does not require Kubernetes deployment artifacts, but it does require the user to fetch the necessary bootstrap information from the Cloud Pak for Watson AIOps UI after connection creation is completed with the remote orchestration option.
- Micro-edge configuration files are present in the container at run time in the following location, which is volume-mounted by the bootstrap script and needs to be watched for updates while the program runs
```
$ ls /bindings/grpc-bridge/
ca.crt  client-id  client-secret  host  id  port  tls.crt  tls.key
```
- In order for the bootstrap script to provide the container image location to download and run on a remote VM, microedgeconfiguration CR needs to added as per the CR specification to bundleManifest artifacts prereqs folder.
