# Covesa Vehicle Signal Specification (VSS) to Drogue Mapping

The COVESA [VSS](https://covesa.github.io/vehicle_signal_specification/) is a data definition format for vehicle signals. The format allows describing different components of a vehicle, how they relate to each other, and the datatypes used for the data. It _does not_ specify a wire format.

The specification is organized by the COVESA project, which is a standardization effort driven by car manufacturers.

## Motivation

Drogue Dopplegänger is a digital twin implementation for Drogue IoT. It allows defining a digital representation of a physical device. This approach could also make sense for a vehicle, where the different signals are modeled in the digital twin system.

To support automotive use cases, defining a mapping from VSS to Drogue Dopplegänger would make Drogue IoT even more relevant for automotive, and would also be a good test of Drogue IoT's extensibility.

## Scope

There are multiple ways to build VSS on top of Drogue IoT, but in this document we will focus on the following:

* Mapping VSS signal specification to the Drogue Dopplegänger metadata.
* Mapping VSS signal _values_ to the Drogue Dopplegänger data. This allows the state of a vehicle to be represented as a digital twin. The data is represented using [VISSv2 core data representation](https://raw.githack.com/w3c/automotive/gh-pages/spec/VISSv2_Core.html#data-representation), but this may change.

NOTE: Using Drogue Cloud as a connectivity layer is out of scope for this RFC. Defining mapping of the [VISSv2 transport layer](https://www.w3.org/TR/viss2-transport/) using Drogue Cloud is a future task.

## Outcome

The initial outcome of this work is a tool that can read the VSS files (.vspec) and generate the Dopplegänger resources to represent it in a digital twin.

## Terminology

A _Thing_ is a digital representation in Drogue Dopplegänger that may represent a sensor or some value or a relationship to other things.

# VSS and Digital Twin

Mapping a VSS specification to digital twin is done in the following manner:

* Every node is represented as a _Thing_.
* Branch nodes contain references to their parents and children
* Attributes, sensor and actuator nodes contain data according to schema as defined by VISSv2 data specification.

The vss-tools is a set of tools that allow generating a JSON schema of a vehicle specification. The idea is to extend this tool to also generate Thing definitions with the appropriate metadata and schema.

NOTE: VSS does not specify the wire format of the data. There is however a definition of that from VISSv2, which we will use for now.

## Example

For the given VSS spec:

```
Vehicle:
  type: branch
  description: High level vehicle data
Vehicle.Speed:
  datatype: float
  type: sensor
  unit: km/h
  description: Vehicle speed.
Vehicle.Cabin:
  type: branch
  description: All in-cabin components, including doors.
Vehicle.Cabin.DoorCount:
  datatype: uint8
  type: attribute
  default: 4
  description: Number of doors in vehicle.
```

We would end up with the following mapping to Things for a device `mycar`:

```
metadata:
  name: mycar/vehicle
  annotations:
    vss/type: "branch"
    vss/description: "High level vehicle data."
schema:
  json: {
    "version": "draft7",
    "schema": {
      "type": "object"
      "properties": {
        "$children": {
          "type": "array"
        },
        "$parent": {
          "type": "string"
        }
      }
    }
  }
---
metadata:
  name: mycar/vehicle/speed
  annotations:
    vss/unit: "km/h"
    vss/type: "sensor"
    vss/description: "Vehicle speed."
schema:
  json: {
    "version": "draft7",
    "schema": {
      "type": "object",
      "properties": {
        "ts": {
          "type": "string"
        },
        "value": {
          "type": "string"
        }
      }
    }
  }
---
metadata:
  name: mycar/vehicle/cabin
  annotations:
    vss/type: "branch"
    vss/description: "All in-cabin components, including doors."
schema:
  json: {
    "version": "draft7",
    "schema": {
      "type": "object"
      "properties": {
        "$children": {
          "type": "array"
        },
        "$parent": {
          "type": "string"
        }
      }
    }
  }
---
metadata:
  name: mycar/vehicle/cabin/doorcount
  annotations:
    vss/type: "attribute"
    vss/description: "Number of doors in vehicle."
schema:
  json: {
    "version": "draft7",
    "schema": {
      "type": "object",
      "properties": {
        "ts": {
          "type": "string"
        },
        "value": {
          "type": "string"
        }
      }
    }
  }
```

# VISSv2 transport layer

The VISSv2 draft standard describes a payload format and messaging transport mapping for HTTP, WebSocket and MQTT. In the description of MQTT, it is assumed that both the vehicle and cloud app connect to an MQTT broker. 

For Drogue Cloud, we will define a separate transport implementation that maps to the topic structure of Drogue Cloud, which allows using the device registry to store metadata about cars.
