# [Cloud] Digital twin capabilities

Drogue Cloud should provide digital twin capabilities, to make it easier to work with devices on a data layer.

* Issue: https://github.com/drogue-iot/drogue-cloud/issues/164
* PR: https://github.com/drogue-iot/rfcs/pull/13

## Motivation

Currently, Drogue Cloud helps to solve IoT connectivity tasks. In addition to that, it should also help with data.

A common concept to improving working with device data, in the context of IoT, is the concept of
[digital twins](https://en.wikipedia.org/wiki/Digital_twin).

Drogue Cloud should (optionally) provide digital twin capabilities, if the user opts in to this concept.

## Requirements and non-goals

### Optional feature

Using digital twins should be optional on two layers. On the infrastructure layer, as well as on the application layer.

It should still be possible to deploy Drogue Cloud without any digital twin integration if the administrator of the
infrastructure chooses to do so.

At the same time, if the Drogue Cloud instance is deployed with support for digital twins, the user should still need
to opt in to using digital twins.

### Integrate an existing solution

We should integrate an existing solution and not create a new digital twin platform.

In the following section we work under the assumption that we build the integration based on
[Eclipse Ditto](https://www.eclipse.org/ditto/).

For alternatives, see section [Alternative Digital twin projects](#alternative-digital-twin-projects).

### Part of the infrastructure

The integration should be part of the infrastructure. For the end-user, it should be a seamless experience.

### Multi-tenancy

The solution must integrate with the multi-tenancy concept of Drogue Cloud. It must also integrate with the OAuth2
based SSO system.

## Definitions

* Digital twin
  
  > A digital twin is a virtual representation that serves as the real-time digital counterpart of a physical object or process.
  
  – <cite><a href="https://en.wikipedia.org/wiki/Digital_twin" target="_blank">Wikipedia</a></cite>

* SSO – Single Sign-on

## Detailed design

### Deployment

We re-use the existing `drogue-cloud-twin` Helm chart to deploy the solution alongside `drogue-cloud-core`. The digital
twin component becomes part of the Drogue Cloud infrastructure. It still remains an optional component. If you choose
to note deploy the integration, then you are of course missing the functionality.

### Multi-tenancy

Eclipse Ditto uses namespaces, which maps nicely to the concept of "applications" on the Drogue Cloud side. If enabled
in the Drogue Cloud application, it will map to a namespace in the Ditto infrastructure.

#### Namespace naming

There is catch though, Ditto does not allow the same naming pattern that Drogue does. While Drogue Cloud allows
labels ([RFC 1035](https://datatracker.ietf.org/doc/html/rfc1035)) to be used as application,
Ditto allows `(?<ns>|(?:(?:[a-zA-Z]\w*+)(?:\.[a-zA-Z]\w*+)*+))` (also see https://www.eclipse.org/ditto/basic-namespaces-and-names.html).

As the Drogue Cloud device registry (which also manages the applications) is the leading system, this conflicts on
the `-` (dash) character. Which is allowed in Drogue Cloud, but forbidden in Ditto.

There is a discussion in Ditto to lift this requirement (https://github.com/eclipse/ditto/issues/1231). However, if that
doesn't get accepted, we would need an alternative.

The proposal is to translate this into the underscore character, which isn't allowed on the Drogue Cloud side. So simple
translation can be performed:

| Drogue Cloud | Eclipse Ditto |
| --- | --- |
| `foo` | `foo` |
| `foo-bar` | `foo_bar` |

#### Access control



## Breaking changes

## Alternatives

### Ditto namespace mapping

An alternative to the proposed mapping to underscore, it would be possible to use a `.` (dot) instead:

| Drogue Cloud | Eclipse Ditto |
| --- | --- |
| `foo` | `foo` |
| `foo-bar` | `foo.bar` |

### Alternative Digital twin projects

* ?