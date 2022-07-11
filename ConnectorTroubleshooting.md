# Connector troubleshooting

Connectors operate asynchronously, sometimes from a remote location. Find help here for determining the
cause of common problems and solutions.

## Logging

Logging levels for OpenLiberty containers can be modified by following the OpenLiberty
[documentation](https://openliberty.io/docs/22.0.0.3/log-trace-configuration.html#configuaration).
In practice, this generally means either modifying the `/config/server.xml` file directly, or the creating the
`/config/configDropins/overrides/server.xml` override file with the required log level. For example,
```xml
<server>
    <logging consoleFormat="simple"
        consoleSource="message,trace"
        consoleLogLevel="info"
        traceFileName="stdout"
        traceFormat="BASIC"
        traceSpecification="com.ibm.aiops.connectors.*=all" />
</server>
```

## Deployment errors

### BundleManifest is in a Retrying State

When a problem is discovered retrieving or by using the bundle that is referenced by the BundleManifest resource, the status
shows that it is either in an errored or retrying state.

If the RepositoryReady status condition on the BundleManifest resource is failing, this might indicate that the
repository might not be pulled. Either the repository does not exist, or it cannot be accessed due to a network or authentication error. Refer to the BundleManifest documentation to address this.

If the DeployablesReady condition is failing, then the resources are failing to deploy. Check the Kubernetes events for
the BundleManifest and GitApp resources. These can be observed on the Red Hat OpenShift events page, or by querying the
Kubernetes api: `oc get events`. These events contain more detailed information that can be used to determine the
problem.

### GitApp event contains: Attempted to Overwrite Resource

This event indicates that the GitApp failed to install or update because it conflicts with the resources that are installed
by another GitApp. For the two to be compatible, the conflicting resource needs to be renamed in one of the
bundles.

## Communication errors between the Connector and the Cloud Pak for Watson AIOPs Server

The following sections address communication errors that users might run into.

### Connector component has Unknown status

If the component phase is Unknown and the reason that is given is that a Cloud Event was not acknowledged,
this indicates that status updates are not being received.

```
status:
  components:
    connector:
      observedGeneration: 1
      phase: Unknown
      requeueAfter: 30000000000
      resources:
        error: >-
          Post "https://connector-bridge.cp4waiops.svc:9443/v1/async": context
          deadline exceeded
        reason: >-
          https://connector-bridge.cp4waiops.svc:9443/v1/async did not
          acknowledge Cloud Event
        summary: >-
          unable to determine status of Connector component, connection may have
          been interrupted
```

Verify that the connector is running, either remotely or locally. If it is, and the logs show
it is working correctly, verify that the connector has code to periodically resend its status at least one time every
5 minutes.

### TLS errors

The following exception indicates that the connector was unable to validate the server certificate.
```
[4/7/22, 15:34:57:716 UTC] 0000003c StandardConne W   configuration stream terminated with an error
                                 io.grpc.StatusRuntimeException: UNAVAILABLE: io exception
Channel Pipeline: [SslHandler#0, ProtocolNegotiators$ClientTlsHandler#0, WriteBufferingAndExceptionHandler#0, DefaultChannelPipeline$TailContext#0]
    ...
Caused by: javax.net.ssl.SSLHandshakeException: PKIX path validation failed: java.security.cert.CertPathValidatorException: signature check failed
    ...
Caused by: sun.security.validator.ValidatorException: PKIX path validation failed: java.security.cert.CertPathValidatorException: signature check failed
    ...
Caused by: java.security.cert.CertPathValidatorException: signature check failed
    ...
Caused by: java.security.SignatureException: Signature does not match.
    ...
```

This can happen if the server certificate is refreshed and the connector still trusts the old certificate. If deployed
by using the microedge script, then redownload the script and run it to update the certificates. If deployed on the
server, and the connector does not automatically detect updates to certificates, the pod might need to be restarted.

If the error is still present, verify that the `tls.crt` entry in the `connector-bridge-connection-info` secret on
the Red Hat OpenShift cluster that runs the server matches the certificate that is used by the connector. If it does, then the
connector is likely the victim of a man-in-the-middle attack as someone is attempting to intercept the requests between
the server and the connector.

### Authentication errors

The following exception indicates that the connector failed to authenticate with the server.
```
[4/7/22, 16:47:38:047 UTC] 00000047 StandardConne W   configuration stream terminated with an error
                                 io.grpc.StatusRuntimeException: UNAUTHENTICATED: unable to authenticate client, invalid client_id or client_secret in encoded credentials
    ...
```

This can happen if the connector credentials are revoked. If deployed by using the Micro edge script, then redownload the
script and run it to update the credentials. If deployed on the server, then the credentials are automatically
re-created when the problem is detected.

### Connection frequently dies

The following exception indicates that communication between the connector and server was stopped unexpectedly.
```
[4/7/22, 17:10:59:264 UTC] 0000004b StandardConne W   configuration stream terminated with an error
                                 io.grpc.StatusRuntimeException: UNAVAILABLE: Network closed for unknown reason
    ...
```

If this happens frequently, and both the connector and server are working, this might indicate a problem with either
the network or a firewall. For example, the firewall can be set up to stop connections older than a minute. This
might degrade performance as the connector needs to reconnect to the server and resend unreceived events.

### Connector is stuck waiting for configuration

The following log message (FINE level) indicates that the connector is establishing a configuration stream with the
server:
```
[4/7/22, 17:10:59:264 UTC] 00000042 StandardConne 1 sending configuration: event={...}
```

If the connector never receives configuration from the server, then it is stuck in a waiting state. This problem
can be seen in development if multiple users are using the same connection. To resolve the problem, each developer uses their own connection. This can also be seen whether the connector component name defined in the ConnectorSchema
does not match the component name that is used by the connector. In that case, change the ConnectorSchema and connector
code so that they match.

## Performance issues

The connector can have a metrics endpoint available at either `/h/metrics` or `/metrics` that can be used to
determine the cause of many performance issues. If deployed on an Red Hat OpenShift cluster and the connector has a PodMonitor
or ServiceMonitor setup, [user workload](https://docs.openshift.com/container-platform/4.10/monitoring/enabling-monitoring-for-user-defined-projects.html)
monitoring can be enabled to periodically scrape this endpoint. The user can then issue queries through the Red Hat OpenShift
Monitoring UI. Below are some useful metrics common to many connectors. Connectors can have metrics specific to the
connector as well.

**Note:**
If metrics are not showing up for a connector that indicates that either the connector is not correctly configured to output metrics
or that there is a problem with [user workload monitoring](https://docs.openshift.com/container-platform/4.10/monitoring/enabling-monitoring-for-user-defined-projects.html)

| Metric Name | Description |
| --- | --- |
| connectors_sdk_runthread_starts_total | the number of times the connector run thread is started |
| connectors_sdk_configurationstream_starts_total | the number of times the configuration stream thread is started |
| connector_sdk_produce_starts_total | the number of times a produce stream thread is started |
| connector_sdk_consume_starts_total | the number of times a consume stream thread is started |
| connectors_sdk_connectorexceptions_thrown_total | the number of exceptions thrown by the connector |
| connectors_sdk_configuration_processtime_seconds_count | the number of times the connector attempted to configure or reconfigure itself |
| connectors_sdk_configuration_processtime_seconds_sum | aggregate time that is spent in configuration or reconfiguration |
| connectors_sdk_configuration_processtime_seconds_max | maximum time spent in a single attempt to configure or reconfigure |
| connectors_sdk_action_processtime_seconds_max | maximum time spent processing an individual event that is received from a consume stream |
| connectors_sdk_action_processtime_seconds_count | number of consume stream messages that are processed |
| connectors_sdk_action_processtime_seconds_sum | aggregate time spent processing consume stream messages |
| connector_sdk_consume_received_total | number of consume stream messages received |
| connector_sdk_consume_dropped_total | number of invalid consume stream messages dropped |
| connector_sdk_produce_sent_total | number of cloud events that are sent to the server |
| connector_sdk_produce_verified_total | number of cloud events the server has verified as being received |
| connector_sdk_producer_badevents_total | the number of sent events without a destination that have been dropped |
| connector_sdk_produce_dropped_total | the number of cloud events dropped for being too large, or because the server rejects them |
| connector_sdk_status_failed_total | the number of times status failed to be sent to the server |
| connector_sdk_status_sent_total | the number of times status was sent to the server |
| connectors_sdk_vault_lookup_duration_seconds_max | maximum time spent performing a vault lookup |
| connectors_sdk_vault_lookup_duration_seconds_count | the number of vault lookups attempted |
| connectors_sdk_vault_lookup_duration_seconds_sum | aggregate time spent performing vault lookups |
| connectors_sdk_vault_lookup_errors_total | the number of errors encountered attempting to lookup a value in vault |
| connectors_sdk_vault_renewal_duration_seconds_max | maximum time spent performing a vault token renewal |
| connectors_sdk_vault_renewal_duration_seconds_count | the number of vault token renewals attempted |
| connectors_sdk_vault_renewal_duration_seconds_sum | aggregate time spent renewing vault tokens |
| connectors_sdk_vault_renewal_errors_total | number of errors encountered attempting to renew vault tokens |

Some example PromQL queries for a connector with id `dabc7a3f-9c44-4505-8890-58907297cd7b`:
```
sum by (channel_name)(irate(connector_sdk_produce_sent_total{connector_id="dabc7a3f-9c44-4505-8890-58907297cd7b"}[1m]) * 30)
```

```
sum by (channel_name)(irate(connector_sdk_produce_verified_total{connector_id="dabc7a3f-9c44-4505-8890-58907297cd7b"}[1m]) * 30)
```

**Note:** the above queries assume that Prometheus was configured with a scrape interval of 30 seconds.

