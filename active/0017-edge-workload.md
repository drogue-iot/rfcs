# [Cloud] Edge workload

* PR: https://github.com/drogue-iot/rfcs/pull/17

## Motivation

In Drogue IoT, we don't have an edge orchestration layer, as we intend to integrate with an existing solution for edge
workload distribution and management.

However, we do want to distribute edge workload, and manage this, possibly through Drogue IoT from the cloud side.

### Example use cases

We have an OPC UA server on the edge, which we need to get data out of. We could deploy a small OPC UA connector
on an edge gateway, which connects to the OPC UA server, and forwards the data to the cloud. The same for receiving
commands.

Similar to OPC UA, we could do the same with Bluetooth, translating to GATT services or mesh networks.

In this case, we want to auto-configure some workload, which we do need to achieve this functionality. Drogue Cloud
should be able to derive some workload deployment and request the deployment from an existing edge orchestration
layer.

More concrete, a user configures "OPC UA connectivity" for a device in the Drogue Cloud registry, which results in a
pod spinning up on the edge, pre-configured with both the OPC UA side, and the Drogue Cloud side, streaming data
to Drogue Cloud.

## Requirements and non-goals

* If possible, we want to adopt Kubernetes semantics
* Have a PoC ready with OPC UA and/or BLE
* We do not want to create a general purpose edge orchestration solution, but build on top of an existing one

## Definitions

<dl>
  <dt>Edge</dt>
  <dd>Outside the cloud, located closer to the actual devices.</dd>
  <dt>Edge gateway</dt>
  <dd>Some small scale compute node, running edge workload</dd>
  <dt>Edge workload</dt>
  <dd>
  Workload which needs to run close to the actual devices and which might require additional
  hardware/drivers, like Bluetooth connectivity.
  </dd>
</dl>

## Detailed design

Summing it up, we want to have the following workflow:

* Define some specialized workload (e.g. OPC UA connectivity).
* A reconciliation tool picks it up, and renders Kubernetes workloads (resources) out of that, writing it back to the device specification.
* The Kubernetes reconcilier picks this change up, and forwards the configuration (without further processing) to the edge orchestration layer, which is responsible to deliver this to the edge, and materialize it on the edge. 

### Workload definition

The user defines some use case specific edge workload in the device configuration. Like with the OPC UA example:

```yaml
spec:
  opcUaConnector:
    server: opc.tcp://1.2.3.4:4840
    username: foo
    password: bar
    defaults:
      samplingInterval: 1s
      publishingInterval: 1s
    subscriptions:
      feature1: "ns=2;s=Temperature"
      feature2: "ns=2;s=Pressure"
```

### Deployment plan

The described workload above will be reconciled by an "OPC UA reconciler", into a deployment plan:

```yaml
spec:
  opcUaConnector: ...
  workload:
    kubernetes:
      resources: # <1>
        - apiVersion: apps/v1
          kind: StatefulSet
          metadata:
            name: opcua-connector
          spec:
            selector:
              matchLabels:
                app: opcua-connector
            replicas: 1
            template:
              metadata:
                labels:
                  app: opcua-connector
              spec:
                containers:
                  - name: connector
                    image: ghcr.io/drogue-iot/opcua-connector:0.1.0
                    volumeMounts:
                      - name: configuration
                        mountPath: /etc/agent
                        readOnly: true
            volumes:
              - name: configuration
                secret:
                  secretName: opcua-connector
        - apiVersion: v1
          kind: Secret
          metadata:
            name: opcua-connector
          data:
            config.yaml: …
```
<1> All Kubernetes resources for this device

### Deployment rollout

The Kubernetes reconciler forwards the configuration to an edge Kubernetes orchestration layer, which ensures that
the workload gets deployed. It also reports back the current state of the workload, by updating the
`.spec.workload.kubernetes` field with the last known state.

### Handling conflicts

The reported edge resources will always override the existing workload section. This may trigger another reconcile run,
but that is ok.

The reconciler which creates edge workload must ensure that it updates existing resources. So it doesn't simply
overwrite an existing deployment, but updates whatever is necessary.

## Breaking changes

This RFC brings no breaking changes.

## Alternatives

### Directly write Kubernetes workload

Instead of rendering Kubernetes workload and then actually writing Kubernetes resources in a second step, we would
directly render Kubernetes resources.

Pros:
 * Save on step in the reconciliation process

Cons:
 * Each workload specific reconciler needs to know how to deal with the edge orchestration layer

Pro/Con?:
* Limit access to the Kubernetes infrastructure of the edge device (only the infrastructure, but not the user can configure edge workload)

### The workload spec section

In the example above, the spec section field of the edge workload is `.spec.workload.kubernetes`. This is a clear
indication of what to expect "Kubernetes workload". However, we said that we want to target Kubernetes only, so it
seems a bit redundant.

Still, while we target Kubernetes only, other might want to have additional capabilities. Which we also don't want to
exclude.

Alternatives for this field could be:

* `.spec.kubernetes` - Shorter, easier to handle with the current `dialect!` macro. Doesn't make it clear what "kubernetes" actually means.
* `.spec.workload` - Would limit "workload" to Kubernetes only.

### Don't align on Kubernetes

Instead of aligning on Kubernetes as workload distribution, we could find an alternative.

That may bring some benefits on the overall size of the edge infrastructure deployment. However, that will also make
it much more complicated and specialized.

As we don't want to create out own edge orchestration later, we would need to adopt another deployment model anyway.
However, even on the edge, Kubernetes seems to be most versatile and future-proof model.
