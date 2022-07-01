= [Cloud] Firmware builds

== Motivation

We currently have the capability of delivering firmware updates over the air to any device capable of using the telemetry and command APIs. We also have a protoype for building firmware and storing it in the OCI registry. 

This RFC is about generalizing the firmware build and fitting into Drogue Cloud, so that users of Drogue IoT have a path from source to firmware for all devices, which in turn enables gitops-style management of IoT devices.

== High level Goals

* A schema for specifying firmware build for device firmware
* An API for triggering firmware builds for applications and devices
* A set of pipelines for running firmware builds

When implemented, Drogue Cloud (Drogue Ajour) will have the ability to build and deliver firmware to devices in a CD pipeline.

== Design

The build system will be based on containers Tekton pipelines running on a Kubernetes cluster. The resulting artifacts can be stored in either an Eclipse Hawkbit registry, or a container registry.

Firmware is assumed to be a single artifact that is delivered to the device. Any more complex seperation of firmware need to be implemented at the application level for now.

The design is split into two parts:

* Build configuration - How to configure builds for the firmware served for a device or group of devices
* Build triggering - The API for triggering the build.
* Build pipeline - Running the build.

=== Build configuration

The newly introduced `.spec.firmware` section of `Application` and `Device` is modified to cover build settings in addition to registry settings:

```
spec:
    firmware:
        oci:
            image: mycontainer:mytag
            build:
                # Builder image
                image: device-builder-image:1.0
                # Source for building firmware
                source:
                    git:
                        uri: https://github.com/drogue-iot/drogue-device
                    path: examples/nrf52/microbit/ble
                # Environment variables to pass to builder
                env:
                    - name: MYENV
                      value: foo
                # Arguments to pass to builder
                args:
                    - --release
                # Firmware artifact to publish
                artifact:
                    path: firmware.bin
```

In the same way as for existing firmware configuration, device-level configuration overrides application level configuration. 
For Eclipse Hawkbit, the same schema is used.

The status section reflects the build progress, and is updated based on the underlying tekton pipeline:

```
status:
    firmware:
        conditions:
           type: BuildStatus:
           status: "False"
           reason: RegistryPushFailed
           message: "Failed pushing artifact to registry: foo"
        conditions:
           type: BuildProgress:
           message: "44.2 %"
```


=== API

A new component, `drogue-ajour-api` implements an API for triggering the builds. The API looks like this:

To trigger a new build for an `Application`:

```
POST /api/builds/v1alpha1/apps/{appId}/trigger
```

If the application does not exist or does not have a firmware definition, 404 is returned. Otherwise the build is triggered according to the firmware spec.

To trigger a new build for a `Device`:

```
POST /api/builds/v1alpha1/apps/{appId}/devices/{deviceId}/trigger
```

If the devices does not exist or does not have a firmware definition, 404 is returned. Otherwise the build is triggered according to the firmware spec.


==== Authentication and Authorization 

API requests require a successful authentication using OIDC with the SSO instance configured for the API. Authorization requires
the `trigger-build` claim to be present for the authenticated user in order for the build to be triggered.

Credentials for pushing firmware to the Hawkbit or OCI registry are configured as part of the API deployment, and is passed to the tekton pipelines that runs the build.

=== Running the build

Builds are run as Tekton Pipelines, and consists of three tasks:

1. Fetch source
2. Build firmware artifact
3. Push firmware artifacts

The two first tasks are independent of which registry is used for storing the firmware, whereas the last task depends on the registry. For each registry type, a separate task will be used, and therefore there will in reality be one tekton pipeline definition for each registry type, with some tasks shared.
