# [Cloud] Mappings for Cloud Event Attributes to CoAP Options

The [CoAP protocol](https://datatracker.ietf.org/doc/html/rfc7252) uses [options](https://datatracker.ietf.org/doc/html/rfc7252#section-5.4), allowing the mapping of ["header-like values"](https://datatracker.ietf.org/doc/html/rfc7252#section-12.2) to option numbers ranging between 0-65535. This RFC is for defining the mapping between CoAP Options and Cloud Events.

* PR: https://github.com/drogue-iot/rfcs/pull/8

Also see:

* https://github.com/cloudevents/spec/blob/v1.0.1/spec.md
* https://github.com/cloudevents/spec/blob/v1.0.1/documented-extensions.md
* https://datatracker.ietf.org/doc/html/rfc7252

## Motivation

CoAP uses [Options](https://datatracker.ietf.org/doc/html/rfc7252#section-5.4) instead of the traditional headers we see in other protocols. These options are represented by option numbers ranging from 0-65536, with some options numbers having reserved definitions in the [CoAP specification](https://datatracker.ietf.org/doc/html/rfc7252). 

This document aims to associate specific options to Cloud Event fields (usage defined by [this RFC](https://github.com/pranav-bhatt/rfcs/blob/main/active/0003-cloud-events-mapping.md)).

## Glossary

<dl>
<dt>

**Comments needed**
</dt>

<dd>

**Can be filled once the rest of the document is completed**
</dd>

<dt>CoAP Option Numbers 

**Not Sure if this belongs here**

</dt>
<dd>

According to [Section 12.2 in RFC7572](https://datatracker.ietf.org/doc/html/rfc7252#section-12.2)

> The IANA policy for future additions to this sub-registry is split into three tiers as follows. The range of 0..255 is reserved for options defined by the IETF (IETF Review or IESG Approval. The range of 256..2047 is reserved for commonly used options with public specifications (Specification Required). The range of 2048..64999 is for all other options including private or vendor-specific ones, which undergo a Designated Expert review to help ensure that the option semantics are defined correctly.  The option numbers between 65000 and 65535 inclusive are reserved for experiments. They are not meant for vendor-specific use of any kind and MUST NOT be used in operational deployments.
</dd>

</dl>

## Requirements and non-goals

* Define the association of options to the required and optional fields in a Cloud Event.
* Compare with other projects how they do it.
* This RFC is not about the actual payload, only about the metadata.

## Attributes mapping to CoAP Option Numbers

|         Field         | Required | Option | Description |
| --------------------- | -------- | ------ | ----------- |
|    `Content-Format`   |          |   12   | Defined by [Section 5.10.3 in RFC7572](https://datatracker.ietf.org/doc/html/rfc7252#section-5.10.3). Will be used to fill the `datacontenttype` field in the Cloud Event |
| `Authorization token` |    ✓     |  2048  | This token will be sent to the Authorization service, which will return the [device id](#device-id) upon validation, which will be used to fill the `source` field in the Cloud Event.|
|   `Type Information`  |          |  2049  | Type of Event, as provided by the device |
|     `Data Schema`     |          |  2050  | Schema of the payload |
|       `Subject`       |          |  2051  | Added as Cloud Events sent for cloud-to-device communications include this field |
|        `Time`         |          |  2052  | Added as Cloud Events sent for cloud-to-device communications include this field |