---
title: Metaldata Service

ep-number: 18

creation-date: 2026-03-30

status: implementable

authors:

- "@maxmoehl"

reviewers:

- "@afritzler"
- "@hardikdr"

---

# IEP-18: Metaldata Service

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)

## Summary

Introduce a new component to the IronCore metal-automation which serves metadata
information to servers.

## Motivation

In order to configure a server properly the script configuring it needs to be
aware of which server it is configuring to choose the correct host name, FRR
configuration and other per-server properties.

One way to achieve this is to create per-server ignition (or cloud-init) scripts
which are already tailored to the host they are supposed to run on. However,
sometimes these scripts can not be fully customized as they are re-used for a
set of servers. The workaround so far was to create an inventory file which
contains the per instance data using the system UUID as a key. A startup script
would then look at that inventory file, pick up the right metadata and fill in
the gaps. This is a cumbersome process as it requires collecting all system
UUIDs upfront, the scripts are not reusable across environments, and every
server has access to all the metadata.

To alleviate this issue we propose to introduce a meta(l)data service.

### Goals

* Serve metadata information: hostname, zone, IP prefix.
* Serve user-data in a key-value format.
* Version the endpoints to support breaking changes in the future.

### Non-Goals

`¯\_(ツ)_/¯`

## Proposal

Implement a meta(l)data service.

The metadata should be presented as JSON by default, support for additional
formats may be added later on. The client can request a specific format using
the `Accept` HTTP header. The response should look like this:

```json
{
    "server-name": "cn-wdf4a-1-system-0",
    "hostname": "cn-wdf4a-1",
    "zone": "wdf4a",
    "ip-prefix": "2001:db8::/32",
    "user-data": {
        "foo": "bar"
    }
}
```

### Data Source

The top-level keys are populated based on the annotations and labels of the
server in the following format:

```yaml
metadata.metal.ironcore.dev/$key: $value
```

This allows the operator to specify arbitrary key-value pairs in the returned
metadata JSON. When an annotation and label with the same key are specified, the
label takes precedence.

In addition to those key-value pairs, the name of the server objects is also
included in the field `server-name`. Specifying an annotation or label with
that key overwrites it with the given value.

For data that is common among all servers (e.g. the region or zone) the operator
can define static key-value pairs which are set on every response. Those can
also be overridden by annotations / labels and the key `server-name` cannot be
specified.

Beyond the server annotations / labels a field for user data will be
added. This allows us to progress towards feature-parity with metadata services
offered by similar platforms. The [list of supported
platforms](https://coreos.github.io/ignition/supported-platforms/) for ignition
shows that it is a common pattern to use such a field to deliver the ignition
script. This has the added benefit that we are no longer hard-wired to the
ignition ecosystem as this approach could be adapted to support other systems
like cloud-init.

User-data can be supplied via the `UserDataRef` of a `ServerClaim`. The
metadata server resolves the claim bound to a server to get the secret
referenced by the claim containing the user-data.

### Metaldata API

To retrieve data from a local metadata server, clients rely on DNS to resolve to
the correct local instance:

```
metaldata.ironcore.dev
```

The metadata service uses the client IP to identify the server which called it.
The metadata is extracted from the resources which are populated by the
metal-operator, specifically the `Server` custom resource.

Depending on the path different information is returned:

* `/v1/`: the full metadata JSON, similar to the example above.
* `/v1/{field}`: the content of the named field as plain text, e.g. `cn-wdf4a-1` if
  the user called `/hostname` in our example.
* `/v1/user-data`: the user-data as JSON.
* `/v1/user-data/{key}`: the value of the user-data key as plain text.

If a key contains special characters not allowed in URLs, standard URL encoding
must be applied to retrieve them.

This allows us to stay compatible with the `ignition.config.url` kernel cmdline
as the user can supply a rendered ignition via the user-data secret and specify
`metaldata.ironcore.dev/v1/user-data/ignition` as the URL in the kernel cmdline.

### Security Considerations

To prevent the most basic types of server-side-request-forgery (SSRF) the
metaldata service requires a special header to be sent by the client to indicate
that it intended to retrieve its metadata:

```
Metadata-Flavor: IronCore Metal
```

The metaldata service assumes that the network over which the request is made is
secure. IP addresses must be trustworthy and only operators have direct access
to the network devices which handle the packets to prevent eavesdropping as the
service does not implement HTTPS.

We explicitly do not implement the approach of IMDSv2 from AWS which uses a
token based flow. This is done to allow for simple consumption in environments
where only very limited tooling is available (like the initrd phase). A future
version may support such a mechanism as an opt-in feature.

### Metal-API Changes

As outlined earlier we will have to add support for a user-data secret to the
`ServerClaim`. This will be done via a local object reference called
`UserDataRef`:

```go
type ServerClaimSpec struct {
	// Power specifies the desired power state of the server.
	// +required
	Power Power `json:"power"`

	// ServerRef is a reference to a specific server to be claimed.
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:XValidation:rule="self == oldSelf",message="serverRef is immutable"
	// +optional
	ServerRef *v1.LocalObjectReference `json:"serverRef,omitempty"`

	// ServerSelector specifies a label selector to identify the server to be claimed.
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:XValidation:rule="self == oldSelf",message="serverSelector is immutable"
	// +optional
	ServerSelector *metav1.LabelSelector `json:"serverSelector,omitempty"`

	// IgnitionSecretRef is a reference to the Secret object that contains
	// the ignition configuration for the server.
	// +optional
	IgnitionSecretRef *v1.LocalObjectReference `json:"ignitionSecretRef,omitempty"`

	// Image specifies the boot image to be used for the server.
	// +required
	Image string `json:"image"`

	// UserDataRef references a Secret object which contains the user-data which
	// will be served via the metaldata service to the server.
	// +optional
	UserDataRef *v1.LocalObjectReference `json:"userDataRef,omitempty"`
}
```

Apart from the API change, no changes to the metal-operator controllers are
required at this time as this information is only for the metaldata service to
act on.

## Alternatives

We stick with what we have, meaning that we only offer ignition and users have
to generate per-server ignitions or find other workarounds like the inventory
system we had. This includes the open security issues where any user of a
server can enumerate the possible system UUIDs to retrieve ignitions including
their secrets.
