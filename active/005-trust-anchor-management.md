# RFC - Trust Anchor support

This document can be considered an initial step in the wider topic of "Device provisioning".

## Overview

The X.509 certificates are versatile. In the context of IoT, they can provide a better and more scalable method to authenticate devices. Drogue Cloud already has support for authentication of devices using X.509 certs, now we have the opportunity to leverage our elegant command-line client - DRG for managing cert and keys.

## Commands

The very first thing we need to decide is a root subcommand. Some options can be

- drg `trust` ...
- drg `trust-anchor` ...
- drg `adm` ...
- your suggestions

Let's go with `trust` for this document.

Next up is the functionality this command adds to drg. We have a `device` entity that is contained in an isolated box called `app`. This app would have its private key, the device generates a certificate signing request, which is signed by the private key. From that point, this device can authenticate using the certificate. This app works as the trust anchor of the device.

`drg trust create app <app_name>`
This command generates a private key, a certificate, signs that certificate with the private key ( self-signed ), and uploads the certificate and public part of the key to the spec of this app.

- `--file <key_file>` - Users can use their own private keys.
- `--algo <algorithm>` - Users can fine tune the certificate generation by choosing the algorithm.
- `--org <organization_name> --cm <comman_name>` - Users can specify the certificate information using flags, or would be prompted to enter it.

We can define a context in drg, this can be useful to decide on the default algorithm to generate the certificates and keys.

`drg trust create device <device_name> --app <app_name> --out <file_path>`
This command requires the private key of the app and signs a new certificate and outputs the file to the specified file path, or it uploads it to the cloud with some encryption. The certificate can be later flashed to the device.

- `--file <csr_file>` - Users can use their own CSR.

`drg trust create all --app <app_name>`
As the number of devices increases, this process becomes tiresome, so with this command certs for all the devices in this app would be generated and stored in the specification. This requires the private key.

`drg trust export device <device_name> --app <app_id> <file_name>`
This command downloads the device certificates from the cloud and stores them locally. Probably, this happens in the flashing process.

`drg trust print app <app_name>` This command displays the list of all the thumbprints ( digest hash ) of all the root CAs.

`drg trust revoke app <app_name>` This command can be used to delete a trust anchor, the user would be prompted with a list of available CA ( if more than one ).

- `--cert <hash_of_cert>` - Users can specify the CA to delete here.
- `--y` - Users can add this flag, to skip the confirmation prompt.

All devices using this certificate would be marked as unauthenticated. As this is a critical operation, users should be prompted with a cryptographic challenge, maybe a random hash with they have to sign using the private key, thus allowing certificate revocation on the successful completion of the challenge.

## Implementation

TODO: Figuring out where to store the private keys.

The specification of each device needs to contain a unique mapping to its physical counterpart, this can be accomplished by storing the MAC Address + Service number of every device or storing the MAC address as a JWT token provided by the server.

The generation of the private keys and certificates would be done using rust-openssl.
