# IBM Cloud Pak for Watson AIOps Connectors

- A connector is an integration between Cloud Pak for Watson AIOps and a particular data source.
- Connectors are packaged as a container image that runs a gRPC client inside and communicates back to the Cloud Pak for Watson AIOps main cluster by using a gRPC channel.
- The preferred programming model is through Open Liberty. However, any other gRPC-enabled programming model (such as Node.js) can also be used - provided it is a supported runtime.

## Connector composition
A Connector is composed of these components

- A connector container
  - Container image that runs a gRPC client, running either in-cluster for local deployment or remotely for micro-edge deployment.

- Processor containers(0..* ) (backend processors, running in-cluster)
  - Each connector must have at least one component that interacts with its collected data, which we call a processor.  Many types of processors are available, depending on their purpose.

- A UI tile (ConnectorSchema YAML)
   - If you are creating a connector for a data source that is not yet supported in Cloud Pak for Watson AIOps, you first need to create a ConnectorSchema.
   - The ConnectorSchema CR serves multiple purposes:
      - It provides content for the UI to render the corresponding form in the Data and tool integrations section of the product
      - It validates completed UI forms

- Kafka topics (0..* YAMLs)

## Cloud Pak for Watson AIOps Bundle Manifests
An AIOps Bundle Manifest is a Custom Resource (YAML) that can contribute different types of artifacts into Cloud Pak for Watson AIOps like connectors, processors, and so on, which can grow in future to include other types.

## High-level steps to create a new connector and add bundleManifest
- Create a connector repo that uses the example template that is located [here](https://github.com/IBM/cp4waiops-connectors-java-template)
- Rename the connector name from `java-grpc-connector-template` to your unique connector name in all the applicable places.
- Connector-sdk grpc-java-sdk is published as a maven artifact that can be used by third-party connector contributors.
- The example template is implemented by using open-liberty/grpc-java-sdk and covers these data types:
    - Event: Connector template to push Kafka events to Event Lifecycle
    - Metric: Connector template to push Kafka events to Metrics Manager
    - Topology: Connector template for topology bridge, which pushes Kafka events to ASM
- Update the logic according to the connector requirements. For example, in the example connect [path](https://github.com/IBM/cp4waiops-connectors-java-template/tree/main/src/main/java/com/ibm/aiops/connectors/template)
- Publish the connector image to a docker registry.
- The example template also includes bundle-manifest artifacts for the connector integration and deployment on the Cloud Pak for Watson AIOps cluster. Create the bundleManifest artifacts for the connector by using the example artifacts. Reference the published image and tag.