# [Cloud] Multi-tenant support

Allow multiple tenants to use the same infrastructure and share common resources and services, while isolating
data at the same time.

* Issue: https://github.com/drogue-iot/drogue-cloud/issues/12
* PR: https://github.com/drogue-iot/rfcs/pull/1

## Glossary

* *Device* – A device, connecting to the system. Might also be a *gateway device*, communicating on behalf of other
  devices (edge node). Might also be another *service*, providing connectivity for either devices
  (e.g. The Things Network, Sigfox).
* *Gateway device* – A device (for example a non IP enabled device) might use another device for sending data to the
  cloud. So there is one device the data originates from, and (optionally a different) device the connection
  originates from. If the data is coming from a different device then the connection, the latter device is called a
  *gateway device*.
* *User* – A person accessing the system. Identified by some kind of credentials. Might also be another service.
  Alternative terms: "account".
* *Tenant* – A construct to isolate, scope devices into a group. Data and configuration is not shared between different
  tenants. Alternative terms: "(device) scope", "namespace". Also see below: [Unresolved topic – Naming](#naming-tenant-vs-namespace-vs-project-vs-application)
* *Instance* – An instance of the whole deployment, serving multiple tenants.
* *Cluster* – A Kubernetes cluster, possibly running other workload as well.

## Motivation

We want to have some kind of isolation layer, to be able to host multiple users (tenants) on the same infrastructure.
Users should not be able to see or interact with the resources of other users. It may however the possible that a
single user has access to multiple scopes. Or that a single scope is accessible by multiple users.

The main goal here is to share resources between tenants and reduce the overall resource usage on the cluster.

For example: Spinning up an HA Kafka cluster would require at least: 3x Zookeeper, 3x Kafka. Consuming these
amounts of resources for a single "free tier tenant", which might only be allowed to create a hand full of devices
in the system, is just not practical. Even for the relatively low-resource protocol endpoints, it would mean that
1.000 free-tier tenants would consume at last 16MiB * 3 (MQTT, HTTP, Auth) * 1.000 = ~46 GiB.

*Side note:* A good portion of this document is about *device identities*. However, this is a pre-requisite and
integral part of multi-tenancy, as the tenant information is an important part of the tenant scope. So I don't think
it makes too much sense to split off this topic, and then re-iterate over it in the context of multi-tenancy.

## Requirements and non-goals

* It must be possible to run multiple instances of the solution in a single Kubernetes cluster. Each instance
  supporting multiple tenants. This would be needed to run multiple versions alongside.
* While it might be a valid use case, there is no requirement to be able to migrate a tenant from instance to another.
* This approach does not support inter-scope/inter-tenant communication by itself.
* Support X.509 client certificate authentication for devices.

### Tenants own their devices

A tenant owns/contains its devices. That means that as soon as a tenant is being deleted, its devices must be deleted
as well.

It is ok to "soft delete" (mark deleted) the devices or tenant first, if the deletion takes more time/operations,
than a simple removal of the entry.

Deleting and re-creating a tenant with the same ID must not bring back the devices previously owned by the tenant.

### Use case: Replace device, keep stable ID

It should be possible to replace a physical device (e.g. device ID `d1`) with a new physical device, keeping the same
ID, but having different credentials (username/password, client certificate).

For example: assuming we have a device `d1`, which has a unique username/password combination of `mac1`/`foo`. The
device  authenticates using the unique username, and the system maps that to the device `d1`. Now the device breaks and
gets replaced by a new physical device, which has a unique username/password combination of `mac2`/`bar`. Still, the
device should continue to be recognized as `d1` by the system.

## Detailed design

### IDs

Both tenants and devices will have a unique ID. Tenants are unique per-instance. Devices are unique per-tenant.

Tenant IDs:
  * Must be DNS labels as defined by [RFC1123](https://tools.ietf.org/html/rfc1123)
  * This is necessary to ensure that we can use this ID for Kubernetes resources as well
  * Also see: https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names

Device IDs:
 * contain at least 1 octet
 * contain at most 255 octets
 * must conform to UTF-8
 * **Note:** While it is possible to name your device "Device 🦄" or "::::", it might not be a wise choice to do so,
   as some APIs might require you to escape non-ASCII characters. Still, technically it is possible. You should also
   think about this in the context of Unicode normalization. The system will not normalize IDs in any step of
   processing.

### Management API / Persistence

For the moment we store the tenant information in the same PostgreSQL instance which we use for devices. We use
a simple foreign key constraint and "cascade deletes" from tenants to devices when deleting tenants. The plan is to
re-iterate over the device registry later anyway to support integrating with other services. So we only make implement
what really is necessary at this point.

We create some basic CRUD HTTP APIs for the tenant and amend the device APIs with the "tenant" information.

At this point, we ignore all RBAC or authorization of operations. We still require the user to authenticate in order
to make changes, re-using the OAuth2 mechanisms that we already started to adopt. This might allow us later to re-use
the OAuth2 information for authorization as well.

### Schema

#### Database tables

~~~sql
CREATE TABLE tenants (
    ID VARCHAR(64) NOT NULL,

    DATA JSONB,

    PRIMARY KEY (ID)
);

CREATE TABLE tenant_aliases (
    TYPE    VARCHAR(32) NOT NULL,  -- type of the alias, e.g. 'id'
    ALIAS   VARCHAR(256) NOT NULL, -- value of the alias, e.g. <id>
    ID      VARCHAR(64) NOT NULL,
    
    PRIMARY KEY (ALIAS),
    FOREIGN KEY (ID) REFERENCES tenants(ID) ON DELETE CASCADE
);

CREATE TABLE devices (
    ID           VARCHAR(256) NOT NULL,
    TENANT_ID    VARCHAR(64) NOT NULL,

    DATA JSONB,

    PRIMARY KEY (ID, TENANT_ID),
    FOREIGN KEY (TENANT_ID) REFERENCES tenants(ID) ON DELETE CASCADE
);

CREATE TABLE device_aliases (
    TYPE        VARCHAR(32) NOT NULL,  -- type of the alias, e.g. 'id'
    ALIAS       VARCHAR(256) NOT NULL, -- value of the alias, e.g. <id>
    ID          VARCHAR(256) NOT NULL,
    TENANT_ID   VARCHAR(64) NOT NULL,

    PRIMARY KEY (ALIAS, TENANT_ID),
    FOREIGN KEY (ID) REFERENCES devices(ID) ON DELETE CASCADE,
    FOREIGN KEY (TENANT_ID) REFERENCES tenants(ID) ON DELETE CASCADE
);
~~~

### Aliases

A tenant or device can be referenced by ID, or looked up using an alias. Aliases must be unique, like the primary key
of the entity.

An alias consists of a type and an ID value. The type is purely informational and used to provider better information
in case of alias conflicts.

The system populates the aliases automatically from the content of the entity with every change. A unique key violation
on the alias is treated as a conflict on the manipulating operation.

A duplicate alias for the targeting the same ID is ignored.

A default `id` type entry with the entity ID is always added:

~~~yaml
tenants:
  - id: tenant1
  - id: tenant2
~~~

Would result in:

~~~
---
TENANTS
---
"ID"
---
"tenant1"
"tenant2"

---
TENANT_ALIASES
---
"TYPE", "ALIAS", "ID"
---
"id", "tenant1", "tenant1"
"id", "tenant2", "tenant2"
~~~

For devices this might look like:

~~~yaml
devices:
  - id: device1
    tenant: tenant1
  - id: device2
    tenant: tenant1
  - id: device1
    tenant: tenant2
~~~

Resulting in:

~~~
---
DEVICES
---
"ID", "TENANT_ID"
---
"device1", "tenant1"
"device2", "tenant1"
"device1", "tenant2"

---
DEVICE_ALIASES
---
"TYPE", "ALIAS", "ID", "TENANT_ID"
---
"id", "device1", "device1", tenant1"
"id", "device2", "device2", tenant1"
"id", "device1", "device1", tenant2"
~~~

Adding, for example, a unique username to a device, might look like this:

~~~yaml
devices:
  - id: device1
    tenant: tenant1
    credentials:
      - username: user1
        password: foo
      - username: mac1
        password: foo2
        unique: true
~~~

~~~
---
DEVICES
---
"ID", "TENANT_ID", "DATA"
---
"device1", "tenant1", {"credentials": [{"username": "user1", …}, {"username": "mac1", "unique": true, …}]}

---
DEVICE_ALIASES
---
"TYPE", "ALIAS", "ID", "TENANT_ID"
---
"id",       "device1", "device1", "tenant1"
"username", "mac1",    "device1", "tenant1"
~~~

A duplicate "local" alias would cause no problem:

~~~yaml
devices:
  - id: device1
    tenant: tenant1
    credentials:
      - username: device1
        password: foo
        unique: true
      - username: device1
        password: foo2
        unique: true
~~~

Would result in the following list:

~~~
---
DEVICE_ALIASES
---
"TYPE", "ALIAS", "ID", "TENANT_ID"
---
"id", "device1", "device1", "tenant1"
~~~

#### Manually defined aliases

The idea of the aliases is to be populated automatically, derived from the entity content. However,it would still be
possible to create an "alias" section in the entity in order to manually created aliases:

~~~yaml
tenants:
  - id: tenant1
    aliases: ["tenant2"]
~~~

Resulting in:

~~~
---
TENANTS
---
"ID"
---
"tenant1"

---
TENANT_ALIASES
---
"TYPE", "ALIAS", "ID"
---
"id", "tenant1", "tenant1"
"id", "tenant2", "tenant1"
~~~

Note that this would only be used when looking up a tenant. When a tenant is directly referenced only `tenant1` would
work. Also, would the lookup process return `tenant1` in both cases.

#### Alias conflicts

Creating those unique aliases might create conflicts. These conflicts are expected and operations creating conflicts
will be rejected by the system.

For example: Creating a device `d1` and creating a device `d2` with a unique username of `d1` would cause a conflict.
So it is not possible to add a unique username of `d2` that already is used otherwise in the system.

As devices are unique within the scope of a tenant, tenants are unique in the scope of an instance. So more care has
to be taken defining aliases for tenants as that may block others from using the ID.

For example: A tenant of "Bar Inc" could create a trust anchor for `O=Foo Inc`, blocking "Foo Inc" from creating an
appropriate trust anchor. There are two possible solutions for this:

1) An approval process is used to ensure that a trust anchor only gets activated (and thus the alias created) after
   a manual control of the trust anchor.
2) A dedicated instance is used so that a user can self manage all aspects of the tenant configuration. 

#### More alias examples

##### Example #1: Custom endpoint hostname

A tenant is configured with a dedicated hostname: `mqtt.my.corp`. That would result in an additional alias:

* `mytt.my.corp`.

##### Example #2: Device subject name

A device's certificate has a subject name of `CN=device1,O=Foo,OU=Bar`. That would result in the additional alias:

* `CN=device1,O=Foo,OU=Bar`

**Note:** The alias would be case-sensitive.

##### Example #3: Unique device username

Assuming you have a device with id `device1`, a username/password secret of `foo/bar` and a unique username/password
of `user1/pass`, then the device would have the following aliases:

* `device1`
* `user1`

**Note:** The combination `foo/bar` does not manifest as an alias, as it is a non-unique username/password credential
specific to this device.

### Credentials / Secrets

Currently, the following types are available:

* Username/password – A device may have a list of username/password combinations. Usernames need not be unique and
  are only valid for a specific device.
* Unique username/password – A device may have a list of username/password combination, for which the username acts as
  an additional ID for the device. This implies that only one device may have the username assigned. Create the same
  entry for a different device will cause an error. Creating an additional entry, with the same unique username but
  a different password for the same device is fine though.
* PSK/Token/Password only – A secret, with no username. This is typically some random byte array.
* X.509 Client certificate – The tenant may have a list of X.509 trust anchors. A possible mode of operation is to
  extract the "subjectDn" of a trust anchor as additional tenant ID. In that case the subject DN of the trust anchor
  (the issuer DN of the client certificate) can be used to look up the client, however that also means that the
  subject DN of the trust anchor must be unique for all tenants. This is why this extraction should be explicitly
  enabled.
  
  The device would not contain (store) the certificate itself. However, some operations might require a way to lookup
  the device implicitly, but information from the certificate (like the subject DN). For that it would still be
  necessary to store that information as part of the device information, so that the alias extraction can find that
  information. An alternative would be to create the device with an ID of the subject DN.

### Authentication

Operations originated by devices, reaching out to endpoints must be authorized. For this it is necessary to
authenticate a device first.

#### Gateways / Services

For devices connecting through a gateway (or service), we authenticate the gateway device only. The actual device is
used by reference.

**TODO:** Should we use the lookup here as well? For example: a lorawan device might get replaced. In this case the
device EUI could be used as an additional alias (`lorawan/device-eui:123` or even as `id:123`).

It will still be checked if the gateway device has the permission to act on behalf of the device.

#### Endpoints

In order to authenticate a device, we would need the following information:

* Tenant
* Device ID (ID of the gateway device if used)
* Secret (one of)
  * (Username)/password
  * PSK
  * X.509 certificate

The authentication service will respond with the outcome and include the devices capabilities in the case of a positive
authentication:

* Device's capabilities
  * Devices that the device may request operations for (especially when acting as a gateway)
  * Operations (publish, subscribe, …)

#### X.509 client authentication

!! **double check** !!

The validation of X.509 client certificates traditionally happens in TLS handshake phase. However, we not always have
all necessary information available at that time. For example, it may be that the tenant does not use a custom
hostname, but transmits the tenant information as an HTTP query parameter instead.

That is why the X.509 client checked for some basic constraints during the TLS handshake validation (like expiration
date). It will be re-checked later on by the authentication service when the full request information is available.

The authentication service must re-check constraints like the expiration information, so the initial validation
during the TLS handshake is actually optional and may be skipped. Still, it allows the request to fail fast if used. 

#### Authentication service

The authentication service will receive a request in the form of:

~~~rust
enum Credential {
  UsernamePassword(String,String),
  PSK(String), // Actually Password(String) 
  Certificate(X509Certificate),
}

struct Request {
  pub tenants: Vec<String>,
  pub device: String,
  pub credentials: Credential,
}
~~~

The authentication service will expand the list of tenants and devices and try to find a match. Assuming the following
request:

~~~yaml
tenants: ["tenant1", "foo"]
device: "device1"
~~~

That would result in trying the following combinations:

~~~yaml
devices:
- tenant: "tenant1"
  device: "device1"
- tenant: "foo"
  device: "device1"
~~~

If a combination fails to authenticate, the authorization service must try the next combination. However, it is possible
for the authorization service to perform the lookup before starting the evaluation. This can be done using a single
`SELECT … ALIAS IN (?)` query, and might result reduced list of tenants/devices, as the aliases might point to the same
entity.

#### Possible oddities (just checking)

* Two devices get created: `d1` and `d2`. The device `d1` gets a unique username of `d2` assigned. Both devices
  have a password of `foo`.
  
  An endpoint tries to authenticate a device through the following request:
  `devices = ['id:d2', 'username:d2'], password = 'foo'`.
  
  That would still end up using device `d2`, as the `id=d2` requirement comes first.

* Two devices get created: `d1` and `d2`. The device `d1` gets a unique username of `d2` assigned. Both devices
  have a password of `foo`.
  
  An endpoint tries to authenticate a device through the following request:
  `devices = ['id:d2', 'username:d2'], password = 'foo'`
  
  That would end up using `d2`, as the `id=d2` comes first. However, it wouldn't be possible to publish as device
  `d1` using the username `d2`. In order to do so, you would need to use `d1` as username, or explicitly provide the
  device id (`d1`) with the username.

#### HTTP endpoint

##### Publish as device

* `POST /<channel>{/<custom>}` with basic auth `<device>@<tenant>` / `<password>`:
  * Tenant ID: `<tenant>`
  * Device ID: `<device>`
  * Credentials: `PSK(<password>)`

* `POST /<channel>{/<custom>}?tenant=<tenant>` with basic auth `<device>` / `<password>`:
  * Tenant ID: `<tenant>`
  * Device ID: `<device>`
  * Credentials: `PSK(<password>)`

* `POST /<channel>{/<custom>}?tenant=<tenant>&device=<device>` with basic auth `<username>` / `<password>`:
  * Tenant ID: `<tenant>`
  * Device ID: `<device>`
  * Credentials: `UsernamePassword(<username>, <password>)`

* `POST /<channel>{/<custom>}?device=<device>` with basic auth `<username>@<tenant>` / `<password>`:
  * Tenant ID: `<tenant>`
  * Device ID: `<device>`
  * Credentials: `UsernamePassword(<username>, <password>)`

* `POST /<channel>{/<custom>}` with client cert:
  * Tenant ID: `<endpoint>` | `<issuerDn>`
  * Device ID: `<subjectDn>`
  * Credentials: `Certificate(Certificate)`

* `POST /<channel>{/<custom>}?tenant=<tenant>` with client cert:
  * Tenant ID: `<tenant>`
  * Device ID: `<subjectDn>`
  * Credentials: `Certificate(Certificate)`

* `POST /<channel>{/<custom>}?tenant=<tenant>&device=<device>` with client cert:
  * Tenant ID: `<tenant>`
  * Device ID: `<device>`
  * Credentials: `Certificate(Certificate)` # might need additional subjectDn validation

* `POST /<channel>{/<custom>}?device=<device>` with client cert:
  * Tenant ID: `<endpoint>` | `<issuerDn>`
  * Device ID: `<device>`
  * Credentials: `Certificate(Certificate)` # might need additional subjectDn validation

##### Publish as gateway device

* `POST /<channel>{/<custom>}?device=<device>` with basic auth `<gateway>@<tenant>` / `<password>`:
  * Tenant ID: `<tenant>`
  * Device ID: `<gateway>`
  * Credentials: `PSK(<password>)`
  * Publish as: `<device>`

* `POST /<channel>{/<custom>}?device=<device>&tenant=<tenant>` with basic auth `<gateway>` / `<password>`:
  * Tenant ID: `<tenant>`
  * Device ID: `<gateway>`
  * Credentials: `PSK(<password>)`
  * Publish as: `<device>`

* `POST /<channel>{/<custom>}?device=<device>&tenant=<tenant>&gateway=<gateway>` with basic auth `<username>` / `<password>`:
  * Tenant ID: `<tenant>`
  * Device ID: `<gateway>`
  * Credentials: `UsernamePassword(<username>, <password>)`
  * Publish as: `<device>`

* `POST /<channel>{/<custom>}?device=<device>&gateway=<gateway>` with basic auth `<username>@<tenant>` / `<password>`:
  * Tenant ID: `<tenant>`
  * Device ID: `<gateway>`
  * Credentials: `UsernamePassword(<username>, <password>)`
  * Publish as: `<device>`

* `POST /<channel>{/<custom>}?device=<device>` with client cert:
  * Tenant ID: `<endpoint>` | `<issuerDn>`
  * Device ID: `<subjectDn>`
  * Credentials: `Certificate(Certificate)`
  * Publish as: `<device>`

* `POST /<channel>{/<custom>}?device=<device>&tenant=<tenant>` with client cert:
  * Tenant ID: `<tenant>`
  * Device ID: `<subjectDn>`
  * Credentials: `Certificate(Certificate)`
  * Publish as: `<device>`

* `POST /<channel>{/<custom>}?device=<device>&tenant=<tenant>&gateway=<gateway>` with client cert:
  * Tenant ID: `<tenant>`
  * Device ID: `<gateway>`
  * Credentials: `Certificate(Certificate)` # might need additional subjectDn validation
  * Publish as: `<device>`

* `POST /<channel>{/<custom>}?device=<device>&gateway=<gateway>` with client cert:
  * Tenant ID: `<endpoint>` | `<issuerDn>`
  * Device ID: `<gateway>`
  * Credentials: `Certificate(Certificate)` # might need additional subjectDn validation
  * Publish as: `<device>`

#### MQTT endpoint

**Note:** Many applications default the MQTT client ID to a random string. That is why we should provide good support
for that behavior. And that is also why we cannot rely on it containing a specific character, like `@`.

##### Device

* Username/password `<device>@<tenant>` / `<password>`, Client ID: `???`, Topic: `<channel>`
  * Tenant ID: `<tenant>`
  * Device ID: `<device>`
  * Credentials: `PSK(<password>)`

* Username/password `<username>` / `<password>`, Client ID: `<device>@<tenant>`, Topic: `<channel>`
  * Tenant ID: `<tenant>`
  * Device ID: `<device>`
  * Credentials: `UsernamePassword(<username>, <password>)`

* Client Cert, Client ID: `???`, Topic: `<channel>`
  * Tenant ID: `<endpoint>` | `<issuerDn>`
  * Device ID: `<subjectDn>`
  * Credentials: `Certificate(<certificate>)`

* Client Cert, Client ID: `<device>@<tenant>`, Topic: `<channel>`
  * Tenant ID: `<tenant>`
  * Device ID: `<device>`
  * Credentials: `Certificate(<certificate>)` # might need additional subjectDn validation
  * **Note:** Can only be validated after the TLS handshake, would however work without TLS SNI. As we cannot trust
    the `@` being authoritative, we would always need to check the other option first. Unless we would explicitly reject
    the `@` character when doing client certs. This might result in false negatives if a random client id would
    contain a `@`. Maybe that can be made configurable.

##### Gateway Device

* Username/password `<gateway>@<tenant>` / `<password>`, Client ID: `???`, Topic: `<channel>/<device>`
  * Tenant ID: `<tenant>`
  * Device ID: `<gateway>`
  * Credentials: `PSK(<password>)`
  * Publish as: `<device>`

* Username/password `<username>` / `<password>`, Client ID: `<device>@<tenant>`, Topic: `<channel>/<device>`
  * Tenant ID: `<tenant>`
  * Device ID: `<device>`
  * Credentials: `UsernamePassword(<username>, <password>)`
  * Publish as: `<device>`

* Client Cert, Client ID: `???`, Topic: `<channel>/<device>`
  * Tenant ID: `<endpoint>` | `<issuerDn>`
  * Device ID: `<subjectDn>`
  * Credentials: `Certificate(<certificate>)`
  * Publish as: `<device>`

* Client Cert, Client ID: `<gateway>@<tenant>`, Topic: `<channel>`
  * Tenant ID: `<tenant>`
  * Device ID: `<gateway>`
  * Credentials: `Certificate(<certificate>)` # might need additional subjectDn validation
  * **Note:** Can only be validated after the TLS handshake, would however work without TLS SNI. As we cannot trust
    the `@` being authoritative, we would always need to check the other option first. Unless we would explicitly reject
    the `@` character when doing client certs. This might result in false negatives if a random client id would
    contain a `@`. Maybe that can be made configurable.

#### Hono compatibility

The overall goal is to implement the HTTP and MQTT APIs of Hono for the authenticated device and gateway operation.
Currently, we do not target compatibility with the unauthenticated operations.

As both MQTT and HTTP APIs differ from our own, we will spin up dedicated, Hono compatible endpoints. So we do not
clash in any way with our own API.

### Data / Events

#### Publishing events

* Each tenant has its own KNative eventing based delivery endpoint. The focus is on this being a Knative `KafkaChannel`.
  In theory, this could be any Knative eventing endpoint. Events get published as a CloudEvent:
  * `subject` – The device ID

* As each tenant has its own endpoint, the tenant information is not part of the event.

* Currently, we do not auto-provision/auto-create Kafka channels/topics. This feature will be added in the upcoming
  device registry work.

* Initially we will use a simple template string approach to generate the name of the service name to contact. Something
  like: `https://{tenant}-kn-channel.drogue-iot.svc.cluster.local`.

* Consumers of the events will get notified by an "exporter". This is future work and for now we continue to
  directly access the Kafka topic. This implies that, for now, no authentication/authorization is being performed.

#### Command and control

**ToDo:** Need Dejan's help.

## Alternatives

### Aliases ambiguities

The idea of the alias scopes is to have better control of the IDs. To allow controlling the lookup processing use
more specific information. However, that might also create some ambiguous situations.

It shouldn't be a big deal for devices, as they are mostly under the control of a tenant. However, for tenants that
might have more impact. So more care must be taken, thinking about the extraction of properties. Because, for example,
a manual alias might block the "issuer DN" of another tenant.

#### strong-scoped aliases

Instead of using:

~~~yaml
- "device1"
- "mac1"
~~~

We could use:

~~~yaml
- "id:device1"
- "username:mac1"
~~~

When performing a lookup, we would actually need to test the full string (or both fields). That would also mean
that we would need to pass in multiple combinations to the auth service. For example, if basic auth is used, that
may be `id:<device>` or `username:<device>`, however that may create some overhead and also create some ambiguities
that are hard to understand.

**Note:** This was the originally used approach in this document.

#### Non-scoped aliases

It would also be possible to drop the "type" information altogether. However, that would make it hard to understand
*why* an alias is blocked. Having that information available would allow better error reporting in the case of
conflicts for management operations.

### Instance per tenant

Technically it would be possible to create an instance per-tenant. This would simplify a lot of things, as it would
be isolated by default. It would still be possible to re-use components like the Kafka topic. However, other components
(like the SSO instance, the PostgreSQL database) would still be duplicated, and multiplied by the number of tenants.

When re-using the same Kafka topic, that would also mean that still a "tenant ID" would need to be introduced, and thus
this would still have an impact in the current implementation.

## Unresolved topics

### Locating the tenant for X.509 certificates

Currently, we specify `endpoint` and `subjectDn` when locating a tenant. That means that we would need to two lookups.

We need to double check if that is ok, and if there is an alternative.

### Naming: "Tenant" vs "Namespace" vs "Project" vs "Application"

The term "tenant" might imply some kind of "user" or "entity". However, the focus is more on the scoping/isolating
of devices.  A user might want to have multiple scopes for devices, that might still make him the same "tenant".

Terms also used in that context are "project" (EnMasse, OpenShift) or "namespace" (Kubernetes).

Namespace might be a more generic term, however it also conflicts with the Kubernetes concept of a namespace.
Then again, Kubernetes is an implementation detail that the user should not be aware of. So the confusion might only
arise inside the Drogue IoT cloud codebase.

"Application" is used by "The Things Network".

### Consuming data

Multiple tenants share the internal Kafka topic (assuming we do not have a per-tenant topic as described in the
"Alternatives" section). This means that we cannot give the tenant access to the Kafka topic (which might be good
for other reasons as well).

However, the user/tenant needs a ways to consume the data from that Kafka topic.

One idea is that provide "integrations", which would for example forward messages to an HTTP endpoint of your choice
using CloudEvents. This would allow filtering out events by tenant.

This is out of scope for this RFC and is planned to be implemented in the future.

