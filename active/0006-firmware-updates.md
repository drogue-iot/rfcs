# Firmware updates

## Glossary

* DFU: Device Firmware Update
* OTA: Over The Air

## Motivation

A common part in any IoT service is the ability to update a fleet of devices with the latest and greatest software. With Drogue IoT, there is an opportunity
to integrate the device side with the cloud side to improve the user experience.

High level goals:

* OTA DFU capability on all platforms supported by drogue-device
* Publish device firmware to drogue-cloud
* Roll out firmware to devices registered in drogue-cloud

As OTA DFU is a critical piece of msot IoT infrastructures, the capabilities will be embedded into the drogue iot ecosystem.

## Requirements

The requirements are subject to change, and reflect the minimal capabilities needed for an initial version of the OTA DFU to be feature complete.

The API in drogue-cloud should allow:

* (User facing) Publish a new firmware version
* (User facing) Update firmware for a group of devices
* (Device facing) Poll/Notify device about a new firmware version
* (Device facing) Fetch the new firmware

Drogue-device needs a bootloader that can:

* Boot firmwares (duh)
* Write new firmware versions to flash while current firmware is running (banking)
* Handle the case where the new firmware is not working (retry, roll back and load previous firmware)

To make DFU easy for firmware writes, drogue-device needs a component that can:

* Be notified of new firmware versions
* Use a network API (such as `TcpStack`) to fetch new firmware in chunks
* Use the bootloader API to write firmware chunks to flash

# Implementation 

* The cloud + protocol side implementation has been made in [drogue-ajour](https://github.com/drogue-iot/drogue-ajour)
* The client side implementation has been made in [drgdfu](https://github.com/drogue-iot/drgdfu)
* The implementation has been made in [drgdfu](https://github.com/drogue-iot/drgdfu)
* The bootloader implementation has been made in [embassy](https://github.com/embassy-rs/embassy/tree/master/embassy-boot)

See the above projects for details on how it works.
