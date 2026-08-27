# Model UUID offer URLs

Status: design sketch

## Context

JAAS does not expose the controller models of its backing Juju controllers as
normal models. A user therefore cannot address an offer in one of those models
using the model information normally available to the Juju client.

An offer URL currently identifies its source model by qualifier and name:

```text
[<source>:]<qualifier>/<model>.<offer>[:<endpoint>]
```

For example:

```text
jaas:admin/controller.database
```

This is unsuitable for a hidden backing controller model. Its owner and model
name are implementation details of the backing controller, and the model is
not present in the model list exposed by JAAS.

## Scope

This proposal changes how an offer URL can identify its source model. It does
not expose backing controller models through `juju models`, make them valid
`juju switch` targets, or make other model-scoped commands work with them.

Applications that have already consumed an offer are also outside the scope of
the change. Their stored offer URLs remain valid and are not rewritten or
migrated.

Requiring the full `<controller>:<qualifier>/<model>.<offer>` form is not part
of the proposal. The source remains optional where it is optional today.

## Proposed URL form

Add a form in which a full model UUID replaces the qualifier and model name:

```text
[<source>:]<model-uuid>.<offer>[:<endpoint>]
```

For example:

```text
jaas:12345678-1234-4abc-8def-123456789abc.database
```

The UUID is the complete model UUID. Abbreviated or prefix matching is not
allowed for offer URLs.

The existing qualifier and model form remains accepted:

```text
jaas:admin/controller.database
```

The UUID form is the canonical form produced for UUID-capable clients. Keeping
the existing form as an accepted input preserves existing scripts, stored
URLs, and user workflows.

### Disambiguation

The syntax already permits a UUID in the model-name position. A v7 endpoint
interprets an unqualified model component that is a syntactically valid UUID as
a model UUID. A qualifier makes the component a model name instead.

Consequently, a model whose literal name looks like a UUID remains addressable
using the qualified legacy form.

## ApplicationOffers v7

Add version 7 of the `ApplicationOffers` facade as the explicit capability
boundary for model UUID offer URLs. The facade version is a capability signal,
not a test of the controller's product version.

The versions have the following contracts:

| Version | Offer URL contract |
| --- | --- |
| v5 | Legacy Juju 3.6 owner and model semantics. |
| v6 | Existing Juju 4 qualifier and model semantics. |
| v7 and later | Accept model UUID offer URLs and return canonical model UUID offer URLs. |

The v7 wire types do not need to differ from v6. The version establishes the
new resolution and result-normalization semantics.

A v7 endpoint must apply the UUID semantics consistently to facade methods
that accept offer URLs, including looking up consume details, showing an
offer, changing offer access, and removing offers. Offer details returned by
creation, listing, finding, showing, and consumption use the UUID form.

Existing v5 and v6 behaviour remains unchanged. This is important because
already-deployed Juju 4 controllers advertise v6 without supporting UUID
resolution.

## Direct controller compatibility

A Juju 4 client checks the best negotiated `ApplicationOffers` facade version.

When the endpoint supports v7 or later, the client sends the UUID URL and lets
the endpoint resolve it.

When the endpoint supports only v5 or v6, the client acts as a compatibility
adapter:

1. Parse the UUID URL and retain it as the canonical URL.
2. Resolve the source model's full UUID to its qualified model name using the
   local client store.
3. Construct the equivalent `<qualifier>/<model>.<offer>` URL.
4. Use that URL for calls to the v5 or v6 offerer.
5. Rewrite offer URLs in returned details back to the retained UUID form before
   displaying them, persisting them, or sending consume details to the
   consumer.

UUID lookup in the client store must be an exact match. The partial UUID
matching supported by interactive model selection must not be reused for this
purpose.

This design assumes that direct consumption already has both the source and
destination controller and model details in the local client store. Translation
does not require a new discovery API. If a pre-v7 endpoint is selected and the
source UUID is absent from the store, the client returns an explicit resolution
error instead of sending a UUID that the endpoint cannot interpret.

The existing v5-specific conversion of v6 qualifier filters remains in place
after URL translation.

### URLs emitted from a Juju 3.6 offerer

A Juju 3.6 source offer record does not persist an offer URL. It stores the
offer UUID, offer name, application details, and endpoints in the source model.
The URL is a representation constructed at an API or client boundary.

The Juju 3.6 `Offer` API returns only a success or error result. The CLI
constructs the displayed URL after the offer is created. A Juju 4 client has
the source model UUID in its local client store at that point and can display
the canonical UUID URL directly.

Juju 3.6 constructs the legacy URL when returning offer details from list,
find, show, and consume-detail operations. Those details also contain the
source model tag and offer name. A Juju 4 client can therefore construct the
canonical URL directly from the returned source model UUID and offer name; it
does not need a reverse local-store lookup for this output conversion.

Canonicalization must happen before the existing API client conversion drops
the source model tag, or the client-side offer details type must retain the
source model UUID. The local client store remains necessary for the reverse
operation: translating a UUID URL supplied as input into the qualified model
form required by a v5 or v6 offerer.

As a result, all offer URLs emitted by a new Juju 4 client during offer
creation, listing, finding, showing, and consumption can use the canonical UUID
form even when the source is a Juju 3.6 controller.

## JAAS compatibility

JAAS exposes `ApplicationOffers` v7 to new Juju 4 clients while retaining v5
and v6 for older clients. A new client therefore sends the UUID URL to JAAS
without trying to find the hidden model in its local store.

JAAS must maintain a mapping from each backing controller model UUID to:

- the backing controller;
- the qualified model name required by a legacy offerer; and
- any information needed to connect and authorize the downstream request.

JAAS negotiates the facade independently on its connection to the backing
controller:

| Backing facade | JAAS behaviour |
| --- | --- |
| v7 or later | Pass the UUID form through and normalize the result. |
| v5 or v6 | Translate the UUID to the backing controller's qualified model name, then normalize the result back to the UUID form. |

The controller alias in the public URL continues to identify JAAS. It does not
expose the name or location of the backing controller.

Older clients negotiating v5 or v6 retain the existing URL semantics. Access
to offers in hidden controller models through the new UUID form consequently
requires a client and JAAS endpoint that negotiate v7.

## Consumption flow

Consumption has two distinct controller interactions:

```text
Juju 4 client
    |
    | GetConsumeDetails(UUID offer URL)
    v
offer source or JAAS
    |
    | canonical consume details
    v
Juju 4 client
    |
    | Consume(canonical consume details)
    v
consumer controller
```

For a direct v5 or v6 offerer, the client translates only the first interaction
to the legacy URL and restores the UUID URL in the returned consume details.
For JAAS, JAAS performs the equivalent downstream translation.

A Juju 3.6 consumer does not have to resolve the offer URL against the source
model. It receives the consume details from the Juju 4 client. Its existing URL
parser accepts a UUID in the model-name position, so no Juju 3.6 consumer
change is required.

This applies when creating a new consumption. An already consumed application
continues to use the URL stored when it was consumed. The proposal neither
depends on changing that URL nor requires existing consumed applications to be
re-consumed.

## Compatibility matrix

| Path | Negotiation and translation | Required changes | Juju 3.6 changes |
| --- | --- | --- | --- |
| Juju 4 client directly uses a Juju 3.6 offerer, then a Juju 4 consumer | The offerer selects v5. The client translates UUID to qualified model using its local store and restores the UUID in consume details. | Juju 4 client compatibility adapter. | None. |
| Juju 4 client directly uses a Juju 4 offerer, then a Juju 3.6 consumer | An updated offerer selects v7 and resolves the UUID. For an older v6 offerer, the client uses the local-store fallback. The client sends canonical consume details to the consumer. | ApplicationOffers v7 in the updated Juju 4 offerer and support in the Juju 4 client. | None. |
| Juju 4 client uses JAAS in front of a Juju 3.6 offerer, then a Juju 4 consumer | Client and JAAS select v7. JAAS selects v5 downstream and translates using its backing-model mapping. | ApplicationOffers v7 and the backing-model adapter in JAAS; v7 support in the Juju 4 client. | None. |
| Juju 4 client uses JAAS in front of a Juju 4 offerer, then a Juju 3.6 consumer | Client and JAAS select v7. JAAS passes UUID through to a v7 offerer or translates for a v6 offerer. | ApplicationOffers v7 and the backing-model adapter in JAAS; v7 support in the Juju 4 client and updated offerer. | None. |

## Compatibility properties

- Old clients continue to negotiate v5 or v6 with updated Juju controllers and
  JAAS.
- New Juju 4 clients use local-store translation with Juju 3.6 controllers and
  older Juju 4 controllers.
- Existing qualified offer URLs remain valid input to v7 endpoints.
- Juju 3.6 source offer records do not require a migration. The UUID URL can be
  derived from the source model tag and offer name at API boundaries.
- Existing consumed applications are unaffected. Their stored offer URLs
  remain valid and are not rewritten.
- No change is required in a Juju 3.6 offerer or consumer.

## Implementation areas

The work is expected to cover:

- registering and implementing `ApplicationOffers` v7 in Juju 4;
- exact UUID classification and resolution at the v7 server boundary;
- client-side UUID-to-qualified-model translation for v5 and v6;
- canonicalization of returned offer and consume-detail URLs;
- all CLI operations that accept or display offer URLs, including offer,
  consume, find, show, grant, revoke, and remove;
- v7 exposure and backing-controller facade negotiation in JAAS; and
- persistent JAAS knowledge of hidden controller model UUIDs and their backing
  qualified names.

## Error behaviour

- A pre-v7 direct endpoint with no exact local-store UUID mapping produces a
  client-side model-resolution error.
- A v7 endpoint that does not know the UUID produces a model-not-found error.
- A syntactically valid UUID is never resolved by prefix.
- JAAS does not fall back to exposing backing controller or model names when a
  UUID cannot be resolved.

## Open implementation decisions

- Whether canonicalization belongs in the shared `ApplicationOffers` API
  client or in a higher command layer that has access to the Juju client store.
- How JAAS persists and updates the backing controller model UUID mapping.
- Whether v7 resolves UUID-shaped filter model fields as well as fields that
  explicitly contain complete offer URLs.
- The precise user-facing error text when a pre-v7 source UUID is absent from
  the local client store.
