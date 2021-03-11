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

## Requirements

The API in drogue-cloud should allow:

* (User facing) Publish a new firmware version
* (User facing) Update firmware for a group of devices
* (Device facing) Poll/Notify device about a new firmware version
* (Device facing) Fetch the new firmware

Drogue-device needs a bootloader that can:

* Boot firmwares (duh)
* Write new firmware versions to flash while current firmware is running (banking)
* Handle the case where the new firmware is not working (retry, roll back and report)

To make DFU easy for firmware writes, drogue-device needs a component that can:

* Be notified of new firmware versions
* Use a network API (such as `TcpStack`) to fetch new firmware in chunks
* Use the bootloader API to write firmware chunks to flash

## Overall architecture

Publishing a new firmware version:

Updating the firmware of a device:

## Drogue Cloud

## Drogue Device
