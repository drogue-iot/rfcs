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

## Definitions

* Digital twin
  
  > A digital twin is a virtual representation that serves as the real-time digital counterpart of a physical object or process.
  
  – <cite><a href="https://en.wikipedia.org/wiki/Digital_twin" target="_blank">Wikipedia</a></cite>

## Detailed design

## Breaking changes

## Alternatives

### Alternative Digital twin projects

?