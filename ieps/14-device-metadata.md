---
title: Device Metadata in Kubernetes Resources

iep-number: 14

creation-date: 2026-04-09

status: draft

authors:

- "@peanball"
- "@maxmoehl"

reviewers:

- "@peanball"
- "@maxmoehl"

---

# IEP-14: Device Metadata in Kubernetes Resources

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)

## Summary

Physical devices such as servers, routers and network switches have physical
attributes such as location, asset tags, serial numbers and other pertinent
information. This proposal defines a set of well-known Kubernetes labels and
annotations under the `metadata.metal.ironcore.dev/*` namespace to represent
this information on custom resources.

## Motivation

Datacenter resource management, grouping and visualization require information
that cannot be auto-determined from the resources themselves. Such information
will need to be provided externally.

Operators need to filter and select resources by physical properties — for
example, listing all servers in a specific location, rack, or of a certain
height. Visualization tools need this information to render datacenter layouts.
The [metaldata service](13-metaldata-service.md) can expose these
labels to servers at runtime.

### Goals

* Define a base set of well-known labels for physical device metadata.
* Use a dedicated namespace (`metadata.metal.ironcore.dev/*`) to separate
  device metadata from operational annotations.
* Enable filtering and selection of resources by physical location and
  properties using Kubernetes label selectors.

### Non-Goals

* Automatic retrieval of metadata at resource creation via lookup. The goal is
  to _write_ information into resources once they exist.
* Track connectivity information between network devices. This will be done in
  a different enhancement proposal.
* Exhaustively model all possible physical location subdivisions (e.g. floor,
  room, aisle). Additional labels can be introduced later if needed.

## Proposal

Some physical properties cannot be auto-detected for resources managed
by IronCore. This proposal provides data center relevant information that
is useful to operators as filterable labels on Kubernetes resources.

The goal is to apply this metadata schema to all Kubernetes resources that
represent physical devices, which includes servers, storage extenders, and
network equipment such as switches and routers.

### Namespace

All device metadata labels and annotations use the namespace:

```
metadata.metal.ironcore.dev/*
```

This separates physical device metadata from operational annotations (e.g.
`metal.ironcore.dev/operation`) and scopes it to the metal domain. The
metaldata service reads labels and annotations from this namespace and can
expose them to servers.

### Labels

Labels are used for metadata that operators need to filter or select on.

| Label | Description | Example |
|---|---|---|
| `metadata.metal.ironcore.dev/location` | User-defined location identifier at whatever granularity fits the environment (site, campus, datacenter, or a combination) | `CITY-BUILDING-FLOOR` |
| `metadata.metal.ironcore.dev/rack` | Opaque rack identifier (may encode row, position, or other site-specific schemes) | `1-12` |
| `metadata.metal.ironcore.dev/shelf` | Position within the rack (bottom-up U number) | `24` |
| `metadata.metal.ironcore.dev/height` | Device height in rack units (U) | `2` |

The `location` label replaces separate site and datacenter labels. Since each
management cluster corresponds to a single physical location, the granularity
of this value is up to the operator — it may be a site code, a datacenter
identifier, or a composite like `FRA-DC5-3F`.

The `rack` label is intentionally an opaque string. Different environments may
use different conventions to uniquely identify a rack (e.g. row and position,
floor and aisle, or a flat identifier). A recommended format is not prescribed
— the value should be whatever uniquely identifies the rack within the
location.

> [!NOTE]
> 1. Since each management cluster corresponds to a single zone / availability
>    zone, an AZ label is not needed — it is implicit from the cluster context.
> 2. Labels like `height` could theoretically be derived from a device type mapping,
>    but this introduces complexity around vendor naming inconsistencies and hardware
>    revisions. Instead, operators are expected to provide explicit values using
>    their own inventory tooling.

### Status Fields

Information that can be retrieved automatically via the BMC or other means is
stored in the resource `status` and does not need to be represented as labels
or annotations:

```yaml
status:
  biosVersion: 'BIOS Date: 08/21/2025 Ver 1.4'
  indicatorLED: "Off"
  manufacturer: Supermicro
  model: SYS-222H-TN
  networkInterfaces:
  - ips:
    - 2001:db8:203::1:1
    macAddress: 8c:91:3a:ad:45:b8
    name: ens1025f0np0
  - ips:
    - 2001:db8:203::2:1
    macAddress: 8c:91:3a:ad:45:b9
    name: ens1025f1np1
  powerState: "Off"
  serialNumber: SN12345
  sku: Some SKU
  state: Available
```

These fields are useful for visualization and operational tooling but are
managed by controllers, not by operators.

### Example

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: Server
metadata:
  labels:
    metadata.metal.ironcore.dev/location: FRA-DC5-3F
    metadata.metal.ironcore.dev/rack: "1-12"
    metadata.metal.ironcore.dev/shelf: "24"
    metadata.metal.ironcore.dev/height: "2"
  annotations:
    metal.ironcore.dev/loopback-address: 2001:db8:203::1
    metal.ironcore.dev/operation: no-ignore-reconciliation
    metal.ironcore.dev/prefix: 2001:db8:203::/64
```

With these labels, operators can use standard Kubernetes selectors:

```bash
# All servers in location FRA-DC5-3F
kubectl get servers -l metadata.metal.ironcore.dev/location=FRA-DC5-3F

# All 2U servers in rack 1-12
kubectl get servers -l metadata.metal.ironcore.dev/rack=1-12,metadata.metal.ironcore.dev/height=2
```

## Alternatives

### External Inventory Tools

Metadata can be stored in external inventory tools such as
[Netbox](https://github.com/netbox-community/netbox). The API equally allows
reading and writing information from and to Netbox.

As Kubernetes is the foundation of IronCore, keeping auxiliary information in
an external tool seems counterproductive. The custom resources representing
real hardware should contain all the necessary information that operators may
need.

### Opaque Location String

Instead of structured labels, a single opaque location string with a
recommended encoding scheme could be used. Parsing would be delegated to
consumers (e.g. visualization tools). This is simpler to define but prevents
the use of Kubernetes label selectors for filtering by individual location
components.

### Exhaustive Location Taxonomy

A comprehensive set of labels covering all possible physical subdivisions
(floor, room, aisle, row, rack, position, etc.) could be defined upfront.
While this would maximize filterability, it introduces complexity that is not
yet justified by current use cases. If environments with more granular location
schemes emerge, additional labels can be introduced incrementally without
breaking existing deployments.
