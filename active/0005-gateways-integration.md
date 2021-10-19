# [Cloud] Gateway integration

## Motivation

Allow integrating devices using different types of gateways. Gateways can be:

* *IoT protocol gateway* - connecting to the cloud as another device and act on behalf other devices.
* *Network server* - exposing it's own integration endpoints for devices
* *Network service/IoT cloud gateways* - Network servers as a service or IoT cloud provider (The Things Network for example)

Drogue cloud needs to provide:
* A Proper mapping and authorization of devices pushing data through gateways
* A way to route commands to integration endpoints of network or cloud gateways

## Referencing gateways

We define a gateway as a regular device in our system with its own id and credentials.

```json
{
    "metadata": {
        "application": "myapp",
        "name": "gateway1"
    },
    "spec": {
        "credentials": {
            "credentials": [
                {
                    "pass": "hey-rodney"
                }
            ]
        }
    }
}
```

When devices are subsequently registered, they can specify one or more gateways they are connecting through.

```json
{
    "metadata": {
        "application": "myapp",
        "name": "device1"
    },
    "spec": {
        "gatewaySelector": {
            "matchNames": ["gateway1"]
        }
    }
}
```

### Labels and selectors

Using Kubernetes style labels and selectors we can group gateways and allow them to be easily referenced by devices.


```json
{
    "metadata": {
        "application": "myapp",
        "name": "gateway2",
        "labels": {
            "group": "my-gateways"
        }
    },
    "spec": {
        "credentials": {
            "credentials": [
                {
                    "pass": "hey-rodney"
                }
            ]
        }
    }
}
```

```json
{
    "metadata": {
        "application": "myapp",
        "name": "device1"
    },
    "spec": {
        "gatewaySelector": {
            "group": "my-gateways"
        }
    }
}
```

### Examples

```
drg create app lora-app
drg create device ttn-service --app lora-app --data '{"credentials": {"credentials":[{ "pass": "foobar" }]}, "ttn": {"payload": "raw"}}'
drg create device kubecon-device --app lora-app --data '{"gatewaySelector": {"matchNames": ["ttn-service"]}}'
```

## Data pushing

While pushing data the authorization process goes as follows:

* Gateway that uses standard HTTP endpoint will use `as` parameter to define exact device it's representing, like

```bash
echo '{"temp":42}' | http --auth 'gateway1@myapp:hey-rodney' POST https://http.sandbox.drogue.cloud/v1/foo?as=device1
```

* The endpoint calls authentication service with `gateway`, `credentials` and `device` trying in a single operation to authenticate the gateway and authorize it to represent the device.
* If it's not authenticated and authorized `403 FORBIDDEN` status will be returned.

* In case of the gateway that don't use standard HTTP endpoint (like TTN for example) the gateway, its credentials and representing device will be parsed from the request and its payload.

## Command Routing

In the opposite direction, we need to provide a way to deliver commands that can connect through different (multiple) gateways.

All commands enter Drogue cloud through `Command endpoint`. At the moment (and this will remain the default behavior) the command endpoint transforms the HTTP request into the cloud event and sends it
to the appropriate Knative broker.

The goal is to provide a way for the command endpoint to directly send commands to external gateway endpoints if that's configured.

### External gateway endpoints

When commands needs to be *pushed* to external gateway endpoint, we define `commands->external` information of the gateway device `spec` section, like

```json
{
    "metadata": {
        "application": "myapp",
        "name": "ttn-eu-service"
    },
    "spec": {
        "credentials": {
            "credentials": [
                {
                    "pass": "hey-rodney"
                }
            ]
        },
        "commands": {
            "commands": [
                {
                    "external": {
                        "endpoint": "https://integrations.thethingsnetwork.org/ttn-eu/api/v2/down/gateway1/gateway1?key=xxx",
                        "headers": {
                            "Authorization": "Basic ...",
                            "Content-Type": "application/json",
                        },
                        "clientCertificate": "cert",
                        "type": "ttn"
                    }
                }
            ]
        }
    }
}
```

Now we can define device for the service as

```json
{
    "metadata": {
        "application": "myapp",
        "name": "ttn-device"
    },
    "spec": {
        "gatewaySelector": {
            "matchNames": ["ttn-service-eu"]
        }
    }
}
```

Command routing process works like this:

* When a *Command endpoint* receives a command it call `lookup` method of the authentication service passing device and its gateway(s) (Same as `authenticate` just don't do credential verification)
* Authentication service will check if app, gateways and device exist and are not disabled
    * The merged configuration will be returned to the command endpoint including the `commands` section.
    * If there's no `commands` info, the command is sent through Knative broker to all internal endpoints.
    * If it contains `commands->external` info, it will try to push the command to the external gateways and report back success/failure to the original caller.

### Multiple gateways

In some cases a device can connect through multiple external gateways, like

```json
{
    "metadata": {
        "application": "services",
        "name": "ttn-us-device"
    },
    "spec": {
        "gatewaySelector": {
            "matchNames": ["ttn-us-east-service", "ttn-us-west-service"]
        }
    }
}
```

In this case we need resolve to which one we should push a command.

To properly solve this use case we need a *device state* service that will keep track of the last gateway/internal endpoint device connected through. This is beyond the scope of this RFC and for now
we can implement a solution that will try to send a command to all specified gateways.

### Examples

```
drg create app lora-app
drg create device device_id --app lora-app --data '{"credentials": {"credentials":[{ "pass": "foobar" }]}, "ttn": {"payload": "raw"}, "commands": {"commands": [{"external": {"endpoint": "https://integrations.thethingsnetwork.org/ttn-eu/api/v2/down/xxx/xxx?key=xxx", "type": "ttn"}}]}}'
```


## Future

* Design and implement *Device state* service that will be able to provide a last endpoint or external gateway device connected through.
* Use *Device state* service to improve command routing (both internally by sending a command only to a specific endpoint or external gateway endpoint).
* There's a type defined for external gateway so we can define proper payload mappers for both data and commands.
