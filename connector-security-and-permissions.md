# Connector Security And Permissions

## Authentication
Cloud Pak for Watson AIOps connectors must authenticate with the Cloud Pak for Watson AIOps grpc server (also known as connector-bridge) by using two concurrent
mechanisms. The first is a client TLS certificate that is shared by all connectors. The second is an OIDC Client-ID and
Client-Secret that is unique to each connection instance. The Connector-SDK automatically bootstraps these credentials
and the Client-ID and Client-Secret are included automatically within the metadata of each individual RPC call in this
format: `Authentication: base64(Client-ID:ClientSecret)`. This unique set of credentials grants the connector
permission to carry out a limited set of actions based on what permissions are configured in the connector's
ConnectorSchema.

## Permissions
The permissions granted to a connector are defined in the `.spec.permissions` section of the ConnectorSchema.

```yaml
permissions:
  openAPIServices:
    - name: my-server-side-service
  channels:
    - name: cp4waiops-cartridge.my-actions
      operations:
        - read
    - name: cp4waiops-cartridge.my-data-ingest
      operations:
        - write
```

### Channels
The eventing channels that a connector can read from or write to are restricted based on the
`.spec.permissions.channels` section of the ConnectorSchema. Any channels that the connector needs to access must
be listed along with a list of operations that need to be permitted (can be "read" and/or "write"). Connectors are
further restricted in their interactions on a _per instance_ basis such that, for a particular Client-ID and Client-Secret,
the connector is only allowed to write an event to a channel if it includes the
[CloudEvent extension attribute](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/primer.md#cloudevent-attribute-extensions)
`connectionid` set to the unique ID assigned to the connection-instance/credential, which may also match the
Client-ID. The Connector-SDK adds this attribute to outbound messages for you automatically. Similarly, when
reading from a channel, only events with the `connectionid` attribute set to match the UID of the
connection-instance/credential is returned to the connector.

### OpenAPI Services
In some cases, connectors might need to interact with
[CloudEvent compatible](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/bindings/http-protocol-binding.md)
HTTP services that are running on the CP4WAIOps server. Services that the connector requires access to must be listed under the
`.spec.permissions.openAPIServices` section of the ConnectorSchema. The provided service name must match the name of
the Kubernetes service that the connector needs access to. The service may also further authenticate the connector and
its permissions by use of more headers that are automatically be added to the request by the connector-bridge
server. These headers are `Client-ID`, `Client-Secret`, and `Authentication`, which includes an OIDC Bearer token
generated automatically using the connector's Client-ID and Client-Secret.

## Credential Encryption
In many cases a ConnectorConfiguration will include a set of credentials that are used to interact with the target technology.
If these credentials are provided directly (as opposed to being accessed via Hashicorp Vault), then you need to provide
configuration to inform the server of which fields of your ConnectorConfiguration need to be encrypted at rest. This
may be done by configuring the `.spec.encryptedFields` field of the ConnectorSchema with an array of
[JSON-Pointers](https://tools.ietf.org/html/rfc6901) pointing to the fields within the ConnectorConfiguration's
`.spec.config` that should be stored as encrypted values. The values are stored by using an AES-GCM encryption scheme and
are decrypted by the connector-bridge before sending the configurations to the connector.

### Example
```yaml
spec:
  encryptedFields:
    - /password
  schema:
    type: object
    properties:
      password: 
        type: string
```
