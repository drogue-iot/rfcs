# (D)TLS-PSK authentication support in Drogue Cloud

Devices want to encrypt the data that traverses the public internet. For TCP based protocols like HTTP and MQTT, Drogue Cloud supports TLS for encryption. For UDP based protocols like CoAP, DTLS is used. 

For the device to trust Drogue Cloud over encrypted connections, it needs to verify the server identity using a certificate authority.

Drogue Cloud also need to authenticate devices. The authentication schemes supported to day are:

* (D)TLS client certificates, where the public key trust of the client is specified according to the [schema](https://github.com/drogue-iot/drogue-cloud/blob/main/console-backend/api/index.yaml#L998).
* Password based authentication, where the password is specified according to the [schema](https://github.com/drogue-iot/drogue-cloud/blob/main/console-backend/api/index.yaml#L945).

## What is TLS-PSK

In TLS 1.3, and DTLS, there exists the possibility to use pre-shared key (PSK). Both sides of the connection are configured with a key, which they use in the handshake to derive the session keys.

The device can give the hint of which 'identity' the server should use to decide which key to use in the handshake.

## Why TLS-PSK?

A standard D(TLS) handshake using certificates consumes a lot of resources on the device side, because the server certificate and certificate authority must be held in memory during the handshake.

TLS-PSK avoids this problem by having the device and cloud share a secret used to verify the peer identity during the (D)TLS handshake.

### Side note: DTLS and CoAP

For CoAP, DTLS combined with a pre-shared key is a common way to authenticate devices, because devices using CoAP are resource constrained and therefore less likely to have the resources to perform CA verification.

## TLS-PSK and Drogue Cloud

TLS-PSK support in Drogue Cloud requires the following changes:

* Extending the schema to allow specifying pre-shared keys for a device
* API in device authentication service for retrieving the pre-shared key for a device
* Extend HTTP, MQTT and CoAP endpoints to allow authenticating devices using TLS-PSK

### Configuration

Devices that wish to authenticate using TLS-PSK must have the following configuration applied:

```yaml
spec:
  credentials:
    - psk:
        key: mysecret
        validity: # Optional
          notBefore: 2022-10-06T09:11:17Z
          notAfter:  2022-10-07T09:11:17Z
```

Pre-shared keys themselves does not have any expiry built in, and for updating keys, a validity range can be set. The terms used for validity originates from the X.509 RFC.

Multiple pre-shared keys may be specified.

### Key selection

Upon authentication a device, the following rules are applied for PSK credentials:

* If no keys with a validity range matching the current time can be found, the device is rejected.
* If multiple valid keys matching the current time can be found, the key selection criteria is in the following order:
   * The most recent key (latest notBefore)
   * The key latest to expire (latest notAfter)
   * The first entry in the list

### Authentication service

Given the nature of TLS-PSK, the unhashed key must be used as input during the handshake. For this, the device authentication service API is extended to provide a pre-shared key for a device, if any exist. The device authentication service shall apply the key selection rules as above for selecting the key to be returned.

### Client configuration

Clients wishing to use TLS-PSK, must provide an identity hint for the service to lookup the pre-shared key. The identity must following the following format:

```
<device_id>@<application>
```

### Endpoints

The HTTP, MQTT and CoAP endpoints, when configured to use OpenSSL (rustls does not support PSK), are configured to support TLS-PSK using an environment variable. If configured with TLS-PSK support, the endpoints will parse the client identity hint and retrieve the pre-shared key from the authentication service.

NOTE: For async runtimes, the rust-openssl crate only provides blocking callback APIs for entering the pre-shared key. This means that each TLS session negotiation must be performed in it's own OS thread to avoid blocking other requests. The endpoints should apply a limit on how many such sessions it can negotiate at the same time to prevent DoS attacks.
