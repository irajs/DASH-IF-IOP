# Content protection and security # {#security}

DASH-IF does not specify a [=DRM system=]. However, DASH-IF provides guidelines for allowing multiple externally defined [=DRM systems=] to protect DASH content by adding encryption signaling and [=DRM system configuration=] to DASH presentations that are encrypted in conformance to Common Encryption ([[!MPEGCENC]]). In addition to content authoring guidelines, DASH-IF specifies interoperable workflows for DASH client interactions with [=DRM systems=], platform APIs and external services involved in content protection interactions.

Common Encryption specifies several [=protection schemes=] which can be applied by a scrambling system and used by different [=DRM systems=]. The same encrypted DASH presentation can be decrypted by different [=DRM systems=] if a DASH client is provided multiple sets of [=DRM system configuration=], either in the MPD or at runtime. The encryption key is identified by a UUID-format string called `default_KID` (or sometimes simply `KID`) and is shared between all DRM systems, whereas the mechanisms used for key acquisition and content protection are largely DRM system specific.

<div class="example">
Example `default_KID` in string format: `72c3ed2c-7a5f-4aad-902f-cbef1efe89a9`
</div>

## HTTPS and DASH ## {#CPS-HTTPS}

Transport security in HTTP-based delivery may be achieved by using HTTP over TLS (HTTPS) as specified in [[!RFC8446]]. HTTPS is a protocol for secure communication which is widely used on the Internet and also increasingly used for content streaming, mainly for protectiing:

* The privacy of the exchanged data from eavesdropping by providing encryption of bidirectional communications between a client and a server, and
* The integrity of the exchanged data against forgery and tampering.

As an MPD carries links to media resources, web browsers follow the W3C recommendation [[!mixed-content]]. To ensure that HTTPS benefits are maintained once the MPD is delivered, it is recommended that if the MPD is delivered with HTTPS, then the media also be delivered with HTTPS.

DASH also explicitly permits the use of HTTPS as a URI scheme and hence, HTTP over TLS as a transport protocol. When using HTTPS in an MPD, one can for instance specify that all media segments are delivered over HTTPS, by declaring that all the `BaseURL`'s are HTTPS based, as follow:

```xml
<BaseURL>https://cdn1.example.com/</BaseURL>
<BaseURL>https://cdn2.example.com/</BaseURL>
```

One can also use HTTPS for retrieving other types of data carried with a MPD that are HTTP-URL based, such as, for example, DRM licenses specified within the `ContentProtection` element:

```xml
<ContentProtection
    schemeIdUri="urn:uuid:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    value="DRMNAME version"
    <drm:License>https://MoviesSP.example.com/protect?license=kljklsdfiowek</drm:License>
</ContentProtection>
```

It is recommended that HTTPS be adopted for delivering DASH content. It should be noted nevertheless, that HTTPS does interfere with proxies that attempt to intercept, cache and/or modify content between the client and the TLS termination point within the CDN. Since the HTTPS traffic is opaque to these intermediate nodes, they can lose much of their intended functionality when faced with HTTPS traffic.

While using HTTPS in DASH provides good protection for data exchanged between DASH servers and clients, HTTPS only protects the transport link, but does not by itself provide an enforcement mechanism for access control and usage policies on the streamed content. HTTPS itself does not imply user authentication and content authorization (or access control). This is especially the case that HTTPS provides no protection to any streamed content cached in a local buffer at a client for playback. HTTPS does not replace a DRM.

## Content encryption and DRM ## {#CPS-encryption-and-drm}

A DASH service MAY offer some or all [=adaptation sets=] in encrypted form, requiring the use of a <dfn>DRM system</dfn> to decrypt the content for playback. The duty of a DRM system is to decrypt content while preventing disclosure of the content key and misuse of the decrypted content (e.g. recording via screen capture).

Encrypted DASH content SHALL use either the `cenc` or the `cbcs` <dfn>protection scheme</dfn> defined in [[!MPEGCENC]]. [=Representations=] in the same [=adaptation set=] SHALL use the same protection scheme. Content in different [=adaptation sets=] MAY use different protection schemes.

`cenc` and `cbcs` are two mutually exclusive encryption modes. DASH content encrypted according to the `cenc` protection scheme cannot be decrypted by a DRM system supporting only the `cbcs` protection scheme and vice versa.

Note: Some modern DRM systems support both schemes. Even when this is the case, clients should not assume concurrent use is possible.

In a DASH presentation, every [=representation=] in an [=adaptation set=] SHALL be encrypted using the same content key (identified by the same `default_KID`).

Note: This means that if [=representations=] use different content keys, they must be in different [=adaptation sets=], even if they would otherwise (were they not encrypted) belong to the same [=adaptation set=]. See also [[#seamless-switching-xas]].

## Content protection constraints for CMAF ## {#CPS-cmaf}

The structure of content protection related information in the CMAF containers used by DASH is largely specified by [[!MPEGCMAF]] and [[!MPEGCENC]]. This chapter outlines some additional requirements to ensure interoperable behavior of DASH clients and services.

[=Initialization segments=] SHOULD NOT contain any `moov/pssh` box and DASH clients SHOULD ignore such boxes when encountered. Instead, `pssh` boxes required for [=DRM system=] initialization are part of the [=DRM system configuration=] and SHOULD be placed in the MPD as `cenc:pssh` elements in [=DRM system=] specific `ContentProtection` descriptors (see [[#CPS-mpd-drm-config]]).

Note: Placing the `pssh` boxes in the MPD has become common for purposes of operational agility - it is often easier to update MPD files than rewrite initialization segments when the default [=DRM system configuration=] needs to be updated. Furthermore, in some scenarios the appropriate set of `pssh` boxes is not known when the [=initialization segment=] is created.

Media segments MAY contain `moof/pssh` boxes to provide updates to [=DRM system=] internal state (e.g. to supply new leaf keys in a key hierarchy). These state updates are transparent to the DASH client. See [[#CPS-default_KID-hierarchy]] for an example.

## Encryption and DRM signaling in the MPD ## {#CPS-mpd}

A DASH client needs to recognize encrypted content and to activate a suitable [=DRM system=], configuring it to decrypt content. The MPD informs a DASH client of the protection scheme used to protect content, identifies the content keys that are used and optionally provides the default [=DRM system configuration=] for a set of [=DRM systems=].

The <dfn>DRM system configuration</dfn> is the complete data set required for a DASH client to activate a single [=DRM system=] and configure it to decrypt content using a single content key. **It is supplied by a combination of XML elements in the MPD and/or data sources available to a DASH client at runtime**. The [=DRM system configuration=] often contains:

* DRM system initialization data in the form of a DRM system specific `pssh` box (as defined in [[!ISOBMFF]]).
* DRM system initialization data in some other DRM system specific form (e.g. `keyids` JSON structure used by [[#CPS-AdditionalConstraints-W3C|W3C Clear Key]])
* The used Common Encryption scheme (`cenc` or `cbcs`)
* `default_KID` that identifies the content key
* License server URL
* [[#CPS-lr-model|Authorization service URL]]

The exact set of values required depends on the DRM system (e.g. what kind of initialization data is required) and what mechanism for content key acquisition is used (e.g. [[#CPS-lr-model|the interoperable license request model]]).

When configuring a [=DRM system=] to decrypt content using multiple content keys, a distinct [=DRM system configuration=] is associated with each content key. Concurrent use of multiple [=DRM systems=] is not considered an interoperable scenario.

Note: In principle, it is possible for the DRM system initialization data to be the same for different content keys. In practice, the `default_KID` is often included in the initialization data so this is unlikely. Nevertheless, DASH clients cannot assume that using equal initialization data implies anything about equality of the DRM system configuration or the content key.

### Protection scheme signaling ### {#CPS-mpd-scheme}

The presence of a `ContentProtection` descriptor with `schemeIdUri="urn:mpeg:dash:mp4protection:2011"` on an [=adaptation set=] informs a DASH client that all [=representations=] in the [=adaptation set=] are encrypted in conformance to Common Encryption and require a [=DRM system=] to provide access. See [[!MPEGDASH]] for the definition of this descriptor.

This descriptor SHALL be present for encrypted [=adaptation sets=]. The `value` attribute SHALL be either `cenc` or `cbcs`, matching the used protection scheme. The `cenc:default_KID` attribute SHALL be present and have a value matching the `default_KID` in the `tenc` box, expressed in UUID string notation.

Note: This document uses the `cenc:` prefix to reference the XML namespace `urn:mpeg:cenc:2013` defined in [[!MPEGCENC]].

<div class="example">
Signaling an [=adaptation set=] encrypted using the `cbcs` scheme and with a key identified by `34e5db32-8625-47cd-ba06-68fca0655a72`.

<xmp highlight="xml">
<ContentProtection
    schemeIdUri="urn:mpeg:dash:mp4protection:2011"
    value="cbcs"
    cenc:default_KID="34e5db32-8625-47cd-ba06-68fca0655a72" />
</xmp>
</div>

### Default DRM system configuration ### {#CPS-mpd-drm-config}

A DASH service SHOULD supply a default [=DRM system configuration=] in the MPD for all supported [=DRM systems=] and all content keys. This enables playback without the need for DASH client customization or additional configuration. Alternatively, this data may be supplied by custom business logic executed at runtime (e.g. to load the values from configuration files or orthogonal data sources).

Any number of `ContentProtection` descriptors MAY be present in the MPD on the [=adaptation set=] level to provide [=DRM system configuration=]. The contents of these descriptors MAY be ignored by the DASH client if overridden by custom business logic or runtime data sources - the [=DRM system configuration=] in the MPD simply provides default values known at content authoring time. Each DRM system specific `ContentProtection` descriptor can contain a mix of XML elements and attributes defined by [[!MPEGCENC|Common Encryption]], the [=DRM system=] author, DASH-IF or any other party.

If a DRM system specific `ContentProtection` descriptor contains the same data in multiple forms then the more generic form SHALL have precedence (e.g. if the license server URL is defined both using `dashif:laurl` and a DRM system specific element, the former is used by DASH clients).

A `ContentProtection` element providing default [=DRM system configuration=] SHALL have the attribute `schemeIdUri="urn:uuid:<systemid>"` that identifies the DRM system, with the `<systemid>` matching a value in the [DASH-IF system-specific identifier registry](https://dashif.org/identifiers/content_protection/). The `value` attribute of the `ContentProtection` element SHOULD contain the DRM system name and version number in a human readable form (for diagnostic purposes).

Note: W3C defines the Clear Key mechanism, which is a "dummy" DRM system implementation intended for client and platform development/testing purposes. **Understand that Clear Key does not fulfill the content and content key protection duties ordinarily expected from a DRM system.** For more guidelines on Clear Key usage, see [[#CPS-AdditionalConstraints-W3C]].

For [=DRM systems=] initialized by supplying `pssh` boxes (as defined in [[!ISOBMFF]]), the `cenc:pssh` element SHOULD be present under the `ContentProtection` descriptor if the value is known at MPD authoring time. The base64 encoded contents of the element SHALL be equivalent to a `pssh` box including its header.

Note: The namespace prefix `dashif:` in this document refers to the XML namespace `https://dashif.org/`.

[=DRM systems=] generally use the concept of license requests as the mechanism for obtaining content keys and associated usage policy (see [[#CPS-license-request-workflow]]). For [=DRM systems=] that use this concept, exactly one `dashif:laurl` element SHOULD be present under the `ContentProtection` descriptor, with the value of the element being the default URL to send license requests to. This URL MAY contain [[#CPS-lr-model-contentid|content identifiers]].

For [=DRM systems=] that require proof of authorization to be attached to the license request in a manner conforming to [[#CPS-lr-model]], exactly one `dashif:authzurl` element SHOULD be present under the `ContentProtection` descriptor, containing the default URL to send authorization requests to (see [[#CPS-license-request-workflow]]).

Issue: Allow multiple URLs for failover? If yes, should be aligned with `<Location>` and `<BaseUrl>` logic where multiple URLs are also accepted. Should also make recommendations for failover logic in that case.

[=DRM system=] specific `ContentProtection` elements that do not provide any [=DRM system configuration=] SHOULD NOT be present in an MPD and MAY be ignored by DASH clients.

The contents of DRM system specific `ContentProtection` elements with the same `@schemeIdUri` SHALL be identical in all [=adaptation sets=] with the same `default_KID`. This means that a [=DRM system=] will treat equally all [=adaptation sets=] that use the same content key.

Note: If you wish to change the default [=DRM system configuration=] associated with a content key, you must update all the instances where the data is present in the MPD. For live services, this can mean updating the data in multiple [=periods=].

<div class="example">
A `ContentProtection` descriptor that provides default [=DRM system configuration=] for a fictional [=DRM system=].

<xmp highlight="xml">
<ContentProtection
  schemeIdUri="urn:uuid:d0ee2730-09b5-459f-8452-200e52b37567"
  value="FirstDRM 2.0">
  <cenc:pssh>YmFzZTY0IGVuY29kZWQgY29udGVudHMgb2YgkXBzc2iSIGJveCB3aXRoIHRoaXMgU3lzdGVtSUQ=</cenc:pssh>
  <dashif:authzurl>https://example.com/tenants/5341/authorize</dashif:authzurl>
  <dashif:laurl>https://example.com/AcquireLicense</dashif:laurl>
</ContentProtection>
</xmp>
</div>

### default_KID defines is the contract for DRM system interactions ### {#CPS-default_KID}

A DASH client interacts with one or more [=DRM systems=] during playback in order to control the decryption of content. Some of the most important interactions are:

* Determining the availability of content keys.
* Communicating with the [=DRM system=] to make content keys available for use.

The scope of each of these interactions is defined by the `default_KID`. Each distinct `default_KID` identifies exactly one content key. The impact of this is further outlined in the chapters describing [[#CPS-client-workflows|DASH client DRM workflows]].

If a DASH client exposes APIs/callbacks to business logic code for the purpose of controlling DRM interactions, it SHALL NOT allow these APIs to associate multiple [=DRM system configurations=] with the same `default_KID`. Conversely, DASH client APIs SHOULD allow business logic to associate different [=DRM system configurations=] with different `default_KIDs`.

When selecting [=adaptation sets=] for playback, a DASH client SHALL determine the required set of content keys based on the `default_KID` values. Upon determining that one or more required content keys (as identified by `default_KID` values) are not available the client SHOULD interact with the [=DRM system=] and request it to make availabe the missing content keys. Clients SHALL explicitly request the DRM system to make available all `default_KIDs` signaled in the MPD and SHALL NOT assume that making one content key from this set available will implicitly make others available.

The DASH client and/or [=DRM system=] MAY batch license requests for different `default_KIDs` (and the respective responses) into a single transaction (for example, to reduce the chattiness of license acquisition traffic).

Note: This optimization might require support from platform APIs and/or [=DRM system=] specific logic from the DASH client, as a batching mechanism is not yet a standard part of DRM related platform APIs.

#### `default_KID` in hierarchical/derived/variant key scenarios #### {#CPS-default_KID-hierarchy}

While it is common that `default_KID` identifies the actual content key used for encryption, a [=DRM system=] MAY make use of other keys in addition to the one signalled by the `default_KID` value but this SHALL be transparent to the client with only the `default_KID` being used in interactions between the DASH client and the [=DRM system=].

<figure>
	<img src="Diagrams/Security/KeyHierarchyAndDefaultKid.png" />
	<figcaption>In a hierarchical key scenario, `default_KID` references the root key and only the sample descriptions reference the leaf keys.</figcaption>
</figure>

In a hierarchical key scenario, `default_KID` identifies the root-level key, not the leaf-level key used to encrypt media samples, and the handling of leaf keys is not exposed to a DASH client. As far as a DASH client knows, there is always only one content key identified by `default_KID`.

This logic applies to all scenarios that make use of additional keys, regardless whether they are based on the key hierarchy, key derivation or variant key concepts. For more information on the background and use cases, see [[#CPS-AdditionalConstraints-PeriodReauth-Implementation]].

### Delivering updates to DRM system internal state ### {#CPS-mpd-moof-pssh}

Some DRM systems support live updates to DRM system internal state (e.g. to deliver new leaf keys in a key hierarchy). These updates SHALL NOT be present in the MPD and SHALL be delivered as `moof/pssh` boxes in media segments.

## DASH-IF interoperable license request model ## {#CPS-lr-model}

The interactions involved in acquiring licenses and content keys in DRM workflows have historically been proprietary, requiring a DASH client to be customized in order to achieve compatibility with specific [=DRM systems=] or license server implementations. This chapter defines an interoperable model to encourage the creation of solutions that do not require custom code in the DASH client in order to play back encrypted content. Use of this model is optional but recommended.

Any conformance statements in this chapter apply to clients and services that opt in to using this model (e.g. a "SHALL" statement means "SHALL, if using this model," and has no effect on implementations that choose to use proprietary mechanisms for license acquisition). The authorization service and license server are considered part of the DASH service.

In performing license acquisition, an interoperable DASH client needs to:

1. Be able to prove that the user has the right to use the requested content keys.
1. Handle errors in a manner agnostic to the specific [=DRM system=] and license server being used.

This license request model defines a mechanism for achieving both goals. This results in the following interoperability benefits:

* DASH clients can execute DRM workflows without project-specific customization.
* Custom code specific to a license server implementation is limited to backend business logic.

These benefits increase in value with the size of the solution, as they reduce the development cost required to offer playback of encrypted content on a wide range of DRM-capable client platforms using different [=DRM systems=], with licenses potentially served by different license server implementations.

### Proof of authorization ### {#CPS-lr-model-authz}

An <dfn>authorization token</dfn> is a [[!jwt|JSON Web Token]] used to prove to a license server that the caller has the right to use one or more content keys under certain conditions. Attaching this proof of authorization to a license request is optional, allowing for architectures where a "license proxy" performs authorization checks transparently to the DASH client.

The structure of the [=authorization token=] is defined by the license server implementation, as the [=authorization token=] needs to express implementation-specific license server business logic parameters that cannot be generalized. Note that the [[!jwt]] and [[!rfc7515]] specifications defining the token data structure specify mandatory fields and processing requirements that must be implemented by [=authorization token=] issuers and processors.

An [=authorization token=] is divided into a header and body, though the distinction between the two is effectively irrelevant and merely an artifact of the [[!jwt|JWT specification]] that defines where some general purpose fields are located.

<div class="example">
JWT headers, specifying digital signature algorithm and expiration time (general purpose fields):

<xmp highlight="json">
{
    "alg": "HS256",
    "typ": "JWT",
    "exp": "1516239022"
}
</xmp>

JWT body with list of authorized content key IDs (a hypothetical license server specific field):

<xmp highlight="json">
{
    "authorized_kids": [
        "1611f0c8-487c-44d4-9b19-82e5a6d55084",
        "db2dae97-6b41-4e99-8210-493503d5681b"
    ]
}
</xmp>

The above data sets are serialized and digitally signed to arrive at the final form of the [=authorization token=]: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImV4cCI6IjE1MTYyMzkwMjIifQ.eyJhdXRob3JpemVkX2tpZHMiOlsiMTYxMWYwYzgtNDg3Yy00NGQ0LTliMTktODJlNWE2ZDU1MDg0IiwiZGIyZGFlOTctNmI0MS00ZTk5LTgyMTAtNDkzNTAzZDU2ODFiIl19.HJA7CGSEHkU9WtcX0e9IEBzhxwoiRxicnaZ5QW5wEfM`
</div>

[=Authorization tokens=] are issued by an authorization service, which is part of a solution's business logic. The authorization service has access to project-specific context that it needs to make its decisions (e.g. the active session, user identification and database of purchases/entitlements). A single authorization service can be used to issue [=authorization tokens=] for multiple license servers, simplifying architecture in solutions where multiple license server vendors are used.

<figure>
	<img src="Diagrams/Security/LicenseRequestModel-BaselineArchitecture.png" />
	<figcaption>Role of the authorization service in DRM workflow related communication.</figcaption>
</figure>

An authorization service SHALL digitally sign any issued [=authorization token=] with an algorithm from the "HMAC with SHA-2 Functions" or "Digital Signature with ECDSA" sets as defined in [[!jwt|the JWT specification]]. The HS256 algorithm is recommended as a highly compatible default, as it is a required part of every JWT implementation. License server implementations SHALL validate the digital signature and reject tokens with invalid signatures or tokens using signature algorithms other than those referenced here.

#### Obtaining authorization tokens #### {#CPS-lr-model-authz-requesting}

To obtain an [=authorization token=], a DASH client needs to know the URL of the authorization service. DASH services SHOULD specify the authorization service URL in the MPD using the `dashif:authzurl` element (see [[#CPS-mpd-drm-config]]).

If no authorization service URL is provided by the MPD nor made available at runtime, a DASH client SHALL NOT attach an [=authorization token=] to a license request. Absence of this URL implies that authorization operations are performed in a manner transparent to the DASH client (see [[#CPS-lr-model-deployment]]).

DASH clients will use zero or more [=authorization tokens=] depending on the number of authorization service URLs defined for the set of all selected content keys. One [=authorization token=] is requested from each distinct authorization service URL. The authorization service URL is specified individually for each [=DRM system=] and content key (i.e. it is part of the [=DRM system configuration=] data set). Services SHOULD use a single [=authorization token=] covering all content keys and [=DRM systems=] but MAY divide the scope of [=authorization tokens=] if appropriate (e.g. different [=DRM systems=] might use different license server vendors that use mutually incompatible license token formats).

Note: Path or query string parameters in the authorization service URL can be used to differentiate between license server implementations (and their respective [=authorization token=] formats).

DASH clients MAY cache and reuse [=authorization tokens=] up to the moment specified in the token's `exp` "Expiration Time" claim (defaulting to "never expires"). DASH clients SHOULD automatically request a new [=authorization token=] if the license server indicates that the [=authorization token=] was rejected (for any reason), even if the "Expiration Time" claim is not present or does not indicate expiration (see [[#CPS-lr-model-errors]]).

Before requesting an [=authorization token=], a DASH client SHALL take the authorization service URL and add or replace the `kids` query string parameter containing a comma-separated list of `default_KID` values obtained from the MPD. This list SHALL contain every `default_KID` for which proof of authorization is requested from this authorization service (i.e. every distinct `default_KID` for which the same URL was specified in `dashif:authzurl`). This modified URL, containing the `kids` query string parameter, will be used for the [=authorization token=] request.

To request an [=authorization token=], a DASH client SHALL make an HTTP GET request to this URL, attaching to the request any standard contextual information used by the underlying platform and allowed by active security policy (e.g. any active HTTP cookies). This data can be used by the authorization service to identify the user and assess the rights of the user.

Note: For DASH clients operating on a web browser platform, effective use of the authorization service may require the authorization service to exist on the same origin as the website hosting the DASH client in order to permit session cookies to be used.

If the HTTP response status code indicates a successful result and `Content-Type: text/plain`, the HTTP response body is the authorization token.

<div class="example">
Consider an MPD that specifies the authorization service URL `https://example.com/Authorize` for the content keys with `default_KID` values `1611f0c8-487c-44d4-9b19-82e5a6d55084` and `db2dae97-6b41-4e99-8210-493503d5681b`.

The generated URL would then be `https://example.com/Authorize?kids=1611f0c8-487c-44d4-9b19-82e5a6d55084,db2dae97-6b41-4e99-8210-493503d5681b` to which a DASH client would make a GET request:

<xmp highlight="xml">
GET /Authorize?kids=1611f0c8-487c-44d4-9b19-82e5a6d55084,db2dae97-6b41-4e99-8210-493503d5681b HTTP/1.1
Host: example.com
</xmp>

Assuming authorization checks pass, the authorization service would return the authorization token in the HTTP response body:

<xmp>
HTTP/1.1 200 OK
Content-Type: text/plain

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImV4cCI6IjE1MTYyMzkwMjIifQ.eyJhdXRob3JpemVkX2tpZHMiOlsiMTYxMWYwYzgtNDg3Yy00NGQ0LTliMTktODJlNWE2ZDU1MDg0IiwiZGIyZGFlOTctNmI0MS00ZTk5LTgyMTAtNDkzNTAzZDU2ODFiIl19.HJA7CGSEHkU9WtcX0e9IEBzhxwoiRxicnaZ5QW5wEfM
</xmp>
</div>

If the HTTP response status code indicates a failure, a DASH client needs to examine the response to determine the cause of the failure and handle it appropriately (see [[#CPS-lr-model-errors]]). DASH clients SHOULD NOT treat every failed [=authorization token=] request as a fatal error - if multiple [=authorization tokens=] are used to authorize access to different content keys, it may be that some of them fail but others succeed, potentially still enabling a successful playback experience. The examination of whether playback can successfully proceed SHOULD be performed only once all license requests have been completed and the final set of available content keys is known. See also [[#CPS-unavailable-keys]].

DASH clients SHALL follow HTTP redirects signaled by the authorization service.

#### Issuing authorization tokens #### {#CPS-lr-model-authz-issuing}

The mechanism of performing authorization checks is implementation-specific. Common approaches might be to identify the user from a session cookie, query the entitlements/purchases database to identify what rights are assigned to the user and then assemble a suitable authorization token, taking into account the license policy configuration that applies to the content keys being requested.

The structure of the [=authorization tokens=] is unconstrained beyond the basic requirements defined in [[#CPS-lr-model-authz]]. Authorization services need to issue tokens that match the expectations of license servers that will be using these tokens. If multiple different license server implementations are served by the same authorization service, the path or query string parameters in the authorization service URL allow the service to identify which output format to use.

<div class="example">
Example authorization token matching the requirements of a hypothetical license server.

JWT headers, specifying digital signature algorithm and expiration time:

<xmp highlight="json">
{
    "alg": "HS256",
    "typ": "JWT",
    "exp": "1516239022"
}
</xmp>

JWT body with list of authorized content key IDs:

<xmp highlight="json">
{
    "authorized_kids": [
        "1611f0c8-487c-44d4-9b19-82e5a6d55084",
        "db2dae97-6b41-4e99-8210-493503d5681b"
    ]
}
</xmp>

Serialized and digitally signed: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImV4cCI6IjE1MTYyMzkwMjIifQ.eyJhdXRob3JpemVkX2tpZHMiOlsiMTYxMWYwYzgtNDg3Yy00NGQ0LTliMTktODJlNWE2ZDU1MDg0IiwiZGIyZGFlOTctNmI0MS00ZTk5LTgyMTAtNDkzNTAzZDU2ODFiIl19.HJA7CGSEHkU9WtcX0e9IEBzhxwoiRxicnaZ5QW5wEfM`
</div>

An authorization service SHALL NOT issue [=authorization tokens=] that authorize the use of content keys that are not in the set of requested content keys (as defined in the request's `kids` query string parameter). An authorization service MAY issue [=authorization tokens=] that authorize the use of only a subset of the requested content keys, provided that at least one content key is authorized. If no content keys are authorized for use, an authorization service SHALL [[#CPS-lr-model-errors|signal a failure]].

Note: During license issuance, the license server may further constrain the set of available content keys (e.g. as a result of examining the device's security level). See [[#CPS-unavailable-keys]].

[=Authorization tokens=] SHALL be returned by an authorization service using [[!rfc7515|JWS Compact Serialization]] (the `aaa.bbb.ccc` format). The serialized form of an [=authorization token=] SHOULD NOT exceed 5000 bytes to ensure that a license server does not reject a license request carrying the token due to excessive header size.

#### Attaching authorization tokens to license requests #### {#CPS-lr-model-authz-using}

[=Authorization tokens=] are attached to license requests using the `Authorization` HTTP request header, signaling the `Bearer` authorization type.

<div class="example">
HTTP request to a hypothetical license server, carrying an [=authorization token=].

<xmp>
POST /AcquireLicense HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImV4cCI6IjE1MTYyMzkwMjIifQ.eyJhdXRob3JpemVkX2tpZHMiOlsiMTYxMWYwYzgtNDg3Yy00NGQ0LTliMTktODJlNWE2ZDU1MDg0IiwiZGIyZGFlOTctNmI0MS00ZTk5LTgyMTAtNDkzNTAzZDU2ODFiIl19.HJA7CGSEHkU9WtcX0e9IEBzhxwoiRxicnaZ5QW5wEfM

(opaque license request blob from DRM system goes here)
</xmp>
</div>

The same [=authorization token=] may be used with multiple license requests but one license request can only carry one [=authorization token=], even if the license request is for multiple content keys.

A DASH client SHALL NOT make license requests for content keys that are configured as requiring an [=authorization token=] (e.g. by the presence of `dashif:authzurl` in the MPD) but for which the DASH client has failed to acquire an [=authorization token=].

### Problem signaling and handling ### {#CPS-lr-model-errors}

Authorization services and license servers SHOULD indicate an inability to satisfy a request by returning an HTTP response that:

1. Signals a suitable status code (4xx or 5xx).
1. Has a `Content-Type` of `application/problem+json`.
1. Contains a HTTP response body conforming to [[!rfc7807]].

<div class="example">
HTTP response from an authorization service, indicating a rejected [=authorization token=] request because the requested content is not a part of the user's subscriptions.

<xmp highlight="json">
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{
    "type": "https://dashif.org/drm-problems/not-authorized",
    "title": "Not authorized",
    "detail": "Your active service plan does not include the channel 'EurasiaSport'.",
    "href": "https://example.com/view-available-subscriptions?channel=EurasiaSport",
    "hrefTitle": "Available subscriptions"
}
</xmp>
</div>

A problem record SHALL contain a short human-readable description of the problem in the `title` field and SHOULD contain a human-readable description of the recommended steps to solve the problem in the `detail` field.

Note: The `detail` field is intended to be displayed to users of a DASH client, not to developers. The description should be helpful to the user whose device the DASH client is running on.

During DRM system activation, it is possible that multiple failures occur. DASH clients SHOULD be capable of displaying a list of error messages to the end-user and SHOULD deduplicate multiple records of the same type (e.g. if an authorization token expires, this expiration may cause failures when requesting 5 content keys but should result in at most 1 error message being displayed).

This chapter defines a set of standard problem types that SHOULD be used to indicate the nature of the failure. Implementations MAY extend this set with further problem types if the nature of the failure does not fit into the existing types.

Issue: Let's come up with a good set of useful problem types we can define here, to reduce the set of problem types that must be defined in solution-specific scope.

#### Problem type: not authorized to access content #### {#CPS-lr-model-errortype-not-authorized}

Type: `https://dashif.org/drm-problems/not-authorized`

Title: Not authorized

HTTP status code: 403

Used by: authorization service

This problem record SHOULD be returned by an authorization service if the user is not authorized to access the requested content keys. The `detail` field SHOULD explain why this is so (e.g. their subscription has expired, the requested content keys are for a movie not in their list of purchases, the content is not available in their geographic region).

The authorization service MAY supply a `href` (string) field on the problem record, containing a URL using which the user can correct the problem (e.g. purchase a missing subscription). If the `href` field is present, a `hrefTitle` (string) field SHALL also be present, containing a title suitable for a hyperlink or button (e.g. "Subscribe"). DASH clients MAY expose this URL and title in their user interface to enable the user to find a quick solution to the problem.

#### Problem type: insufficient proof of authorization #### {#CPS-lr-model-errortype-must-supply-usable-authorization-token}

Type: `https://dashif.org/drm-problems/must-supply-usable-authorization-token`

Title: Not authorized

HTTP status code: 403

Used by: license server

This problem record SHOULD be returned by a license server if the proof of authorization (if any) attached to a license request is not sufficient to authorize the use of any of the requested content keys. The `detail` field SHOULD explain what exactly was the expectation the caller failed to satisfy (e.g. no token provided, token has expired, token is for disabled tenant).

Note: If the authorization token authorizes only a subset of requested keys, a license server should not signal a problem and simply return only the authorized subset.

When encountering this problem, a DASH client SHOULD discard whatever authorization token was used, acquire a new [=authorization token=] and retry the license request. If no authorization service URL is available, this indicates a DASH service or client misconfiguration (as clearly, an [=authorization token=] was expected) and the problem SHOULD be escalated for operator attention.

### Possible deployment architectures ### {#CPS-lr-model-deployment}

The interoperable license request model is designed to allow for the use of different deployment architectures in common use today, including those where authorization duties are offloaded to a "license proxy". This chapter outlines some of the possible architectures and how interoperable DASH clients support them.

The baseline architecture assumes that a separate (largely DRM system agnostic) authorization service exists, implementing the logic required to determine which users have the rights to access which content.

<figure>
	<img src="Diagrams/Security/LicenseRequestModel-BaselineArchitecture.png" />
	<figcaption>The baseline architecture with an authorization service directly exposed to the DASH client.</figcaption>
</figure>

While the baseline architecture offers several advantages, in some cases it may be desirable to have the authorization checks be transparent to the DASH client. This need may be driven by license server implementation limitations or by other system architecture decisions.

A common implementation for transparent authorization is to use a "license proxy", which acts as a license server but instead forwards the license request after authorization checks have passed. Alternatively, the license server itself may perform the authorization checks.

<figure>
	<img src="Diagrams/Security/LicenseRequestModel-TransparentArchitecture.png" />
	<figcaption>A transparent authorization architecture performs the authorization checks at the license server, which is often hidden behind a proxy (indistinguishable from a license server to the DASH client).</figcaption>
</figure>

The two architectures can be mixed, with some DRM systems performing the authorization operations in the license server (or a "license proxy") and others using the authorization service directly. This may be relevant when integrating license servers from different vendors into the same solution.

A DASH client will attempt to contact an authorization service if an authorization service URL is provided. If no such URL is provided, it will assume that all authorization checks (if any are required) are provided by the license server and will not attach any proof of authorization.

### Passing a content ID to services ### {#CPS-lr-model-contentid}

The concept of a content ID is sometimes used to identify groups of content keys. The DRM workflows described by this document do not require this concept to be used but do support it if the solution architecture demands it.

In order to make use of a content ID in DRM workflows, the content ID SHOULD be embedded into authorization service URLs and/or license server URLs (depending on which components are used and require the use of the content ID). This may be done either directly at MPD authoring time (if the URLs and content ID are known at such time) or by custom business logic performing MPD post-processing within the DASH client.

No standard mechanism of embedding the content ID is defined, as a content ID is always a proprietary concept. Likely options include:

* Query string parameters: `https://example.com/tenants/5341/authorize?contentId=movie865343651`
* Path segments: `https://example.com/moviecatalog-license-api/movie865343651/AcquireLicense`

Having embedded the content ID in the URL, all DRM workflows continue to operate the same as they normally would, except now they also include knowledge of the content ID in each request to the authorization service and/or license server.

<div class="example">
Example DRM system configuration with the content ID embedded in the authorization service and license server URLs. Each service may use a different implementation-defined URL structure for carrying the content ID.

<xmp highlight="xml">
<ContentProtection
  schemeIdUri="urn:uuid:d0ee2730-09b5-459f-8452-200e52b37567"
  value="AcmeDRM 2.0">
  <cenc:pssh>YmFzZTY0IGVuY29kZWQgY29udGVudHMgb2YgkXBzc2iSIGJveCB3aXRoIHRoaXMgU3lzdGVtSUQ=</cenc:pssh>
  <dashif:authzurl>https://example.com/tenants/5341/authorize?contentId=movie865343651</dashif:authzurl>
  <dashif:laurl>https://example.com/moviecatalog-license-api/movie865343651/AcquireLicense</dashif:laurl>
</ContentProtection>
</xmp>
</div>

Advisement: The content ID SHOULD NOT be embedded in DRM system specific data structures such as `pssh` boxes, as logic that depends on DRM system specific data structures is not interoperable and often leads to increased development and maintenance costs.

## DRM workflows in DASH clients ## {#CPS-client-workflows}

To present encrypted content a DASH client needs to:

1. [[#CPS-selection-workflow|Select a DRM system that is capable of decrypting the content.]]
1. [[#CPS-activation-workflow|Activate the selected DRM system and configure it to decrypt content.]]
    * During activation, [[#CPS-license-request-workflow|acquire any missing content keys and the licenses that govern their use]].

This chapter defines the recommended DASH client workflows for interacting with DRM systems in these aspects. It is assumed that the platform implements an API similar to [[encrypted-media|W3C Encrypted Media Extensions (EME)]].

### Selecting the DRM system ### {#CPS-selection-workflow}

The MPD describes how content is encrypted, with the `default_KID` values identifying the content keys required for playback, and optionally provides the default DRM system configuration together with a list of candidate DRM systems in the form of `ContentProtection` descriptors.

<div class="example">
An adaptation set encrypted with a key identified by `34e5db32-8625-47cd-ba06-68fca0655a72` using the `cenc` encryption scheme.

<xmp highlight="xml">
<AdaptationSet>
    <ContentProtection
        schemeIdUri="urn:mpeg:dash:mp4protection:2011"
        value="cenc"
        cenc:default_KID="34e5db32-8625-47cd-ba06-68fca0655a72" />
    <ContentProtection
        schemeIdUri="urn:uuid:d0ee2730-09b5-459f-8452-200e52b37567"
        value="FirstDrm 2.0">
        <cenc:pssh>YmFzZTY0IGVuY29kZWQgY29udGVudHMgb2YgkXBzc2iSIGJveCB3aXRoIHRoaXMgU3lzdGVtSUQ=</cenc:pssh>
        <dashif:authzurl>https://example.com/tenants/5341/authorize?mode=acme</dashif:authzurl>
        <dashif:laurl>https://example.com/AcquireLicense</dashif:laurl>
    </ContentProtection>
    <ContentProtection
        schemeIdUri="urn:uuid:eb3841cf-d7e4-4ec4-a3c5-a8b7f9f4f55b"
        value="SecondDrm 8.0">
        <cenc:pssh>ZXQgb2YgcGxheWFibGUgYWRhcHRhdGlvbiBzZXRzIG1heSBjaGFuZ2Ugb3ZlciB0aW1lIChlLmcuIGR1ZSB0byBsaWNlbnNlIGV4cGlyYXRpb24gb3IgZHVl</cenc:pssh>
        <dashif:authzurl>https://example.com/tenants/5341/authorize?mode=emca</dashif:authzurl>
    </ContentProtection>
</AdaptationSet>
</xmp>

The MPD signals that there are two DRM systems that should by default be considered candidates for activation:

* FirstDRM contains a full configuration data set, including the optional `dashif:authzurl`.
* SecondDRM is more sparse, requiring the DASH client to provide the license server URL at runtime (and, [[#CPS-lr-model-authz|if proof of authorization is needed by the license server]], the authorization service URL).

</div>

Note: Neither an initialization segment nor a media segment is required to select a DRM system. The MPD is the only component of the presentation used for DRM system selection.

In addition to the MPD, a DASH client can also execute custom business logic for controlling DRM selection and configuration decisions (e.g. loading license server URLs from configuration data instead of the MPD). This is often implemented in the form of callbacks exposed by the DASH client to an "app" layer in which the client is hosted. It is assumed that when executing any such callbacks, a DASH client attaches relevant contextual data in addition to the parameters specified here, allowing the business logic to make fully informed decisions.

The purpose of the DRM system selection workflow is to select a single DRM system that is capable of decrypting the [=adaptation sets=] selected for playback. The selected DRM system is one that is actually implemented by the media platform and for which the required set of configuration data exists.

Note: The set of [=adaptation sets=] considered here need not be constrained to a single [=period=], potentially enabling seamless transitions to a new [=period=] with a different set of content keys (if supported by the platform API).

When encrypted [=adaptation sets=] are initially selected for playback or when the set of encrypted [=adaptation sets=] selected for playback changes, a DASH client SHOULD execute the following algorithm for DRM system selection:

<div algorithm="drm-selection">

1. Let <var>adaptation_sets</var> be the set of encrypted [=adaptation sets=] selected for playback.
1. Ley <var>default_kids</var> be the set of all distinct `default_KID` values of <var>adaptation_sets</var>.
1. Ley <var>default_kid_schemes</var> be a map of `default_KID -> string`, with the map keys being <var>default_kids</var> and the map values being the Common Encryption schemes signaled by `@value` on the `ContentProtection` descriptors with `@schemeIdUri="urn:mpeg:dash:mp4protection:2011"` for each `default_KID`.
1. Let <var>signaled_system_ids</var> be the set of DRM system IDs for which a `ContentProtection` element is present in the MPD on all entries in <var>adaptation_sets</var>.
1. Provide <var>signaled_system_ids</var> to the business logic component and have it return the desired DRM systems as an **ordered** list <var>desired_system_ids</var>.
    * This enables business logic to establish an order of preference where multiple DRM systems are present. This document does not define any default ordering.
    * This enables business logic to filter our DRM systems known to be unsuitable.
    * This enables business logic to indicate DRM systems not signaled in the MPD.
1. Remove from <var>desired_system_ids</var> any DRM systems that are not supported by the platform.
1. For each <var>system_id</var> in <var>desired_system_ids</var>:
    1. Let <var>supported_default_kids</var> be the values from <var>default_kids</var> for which the DRM system indicates that it supports the Common Encryption scheme indicated for this value by <var>default_kid_schemes</var>.
    1. Let <var>configurations</var> be a map where the map keys are <var>supported_default_kids</var> and the map values are the `ContentProtection` elements from the MPD that match the key and the <var>system_id</var>.
        * If there is no matching `ContentProtection` element in the MPD, the map still contains the entry with an empty value, indicating to business logic executed in the next step that it should provide the value.
        * Enhance the MPD-provided default configuration data with synthesized data where appropriate (e.g. [[#CPS-AdditionalConstraints-W3C|to generate W3C Clear Key initialization data in a format supported by the platform API]]).
    1. Provide <var>configurations</var> to the business logic component for inspection and modification.
        * This enables business logic to override the default DRM system configuration.
        * This enables business logic to inject DRM system configuration if it was not embedded in the MPD.
        * This enables business logic to reject content keys that it knows cannot be used.
    1. Remove any entries with empty values from <var>configurations</var>.
    1. If at least one entry remains in <var>configurations</var>:
        1. Execute the DRM system activation workflow on the selected DRM system, providing <var>configurations</var> as input.
        1. Exit this algorithm.
    1. Continue loop to process next DRM system.
1. If this point is reached, no DRM system was selected and playback of encrypted content is not possible.

</div>

In short, take the list of platform-supported DRM systems, in the order specified by custom business logic, and select for activation the first DRM system for which you have configuration data for at least one content key, filtering out content keys used with unsupported Common Encryption schemes.

Advisement: To determine the supported protection schemes, clients in browsers must assume what the CDM supports as there is no standardized API for probing the platform to determine which Common Encryption proteciton scheme is supported. A bug is open on W3C EME and [a pull request exists](https://github.com/w3c/encrypted-media/pull/392) for the ISOBMFF file format bytestream.

If a DRM system is successfully selected, activation and potentially one or more license requests will follow before playback can proceed. These related workflows are described in the next chapters.

### Activating the DRM system ### {#CPS-activation-workflow}

Once a suitable DRM system has been selected, it must be activated by providing it a list of content keys that the DASH client requests to be made available for content decryption, together with DRM system specific configuration data for each of the content keys. The result of activation is a DRM system that is ready to decrypt all encrypted [=adaptation sets=] selected for playback.

During activation, it may be necessary [[#CPS-license-request-workflow|to perform license requests]] in order to obtain some or all of the content keys. Some of the requested content keys may already be available to the DRM system, in which case no license request will be triggered.

Note: The details of stored content key management and persistent DRM system session management are out of scope of this document - workflows described here simply accept the fact that some content keys may already be available, regardless of why that is the case or what operations are required to establish content key persistence.

Once a suitable DRM system [[#CPS-selection-workflow|has been selected]], a DASH client SHOULD execute the following algorithm to activate it:

<div algorithm="drm-activation">

1. Let <var>configurations</var> be the input to the algorithm; it is a map with the entry keys being `default_KID` values identifying the content keys and the entry values being the DRM system configuration data to use for that particular content key.
1. Let <var>pending_license_requests</var> be an empty set.
1. For each <var>kid</var> and <var>config</var> pair in <var>configurations</var> invoke the platform API to activate the selected DRM system and signal it to make <var>kid</var> available for decryption, passing the DRM system any relevant configuration data stored in <var>config</var> (e.g. the `pssh` box).
    * If the DRM system indicates that one or more license requests are needed, add any license request data provided by the DRM system and/or platform API to <var>pending_license_requests</var>, together with the associated <var>kid</var> and <var>config</var> values.
1. If <var>pending_license_requests</var> is not an empty set, execute the [[#CPS-license-request-workflow|license request workflow]] and provide this set as input to the algorithm.
1. Inspect the set of content keys the DRM system indicates are now available and deselect from playback any [=adaptation sets=] for which the content key has not become available.
1. Inspect the set of remaining [=adaptation sets=] to determine if a sufficient data set remains for successful playback. Raise error if playback cannot continue.

</div>

For historical reasons, platform APIs often implement DRM system activation as a per-content-key operation. Some APIs and DRM system implementations may also support batching all the content keys into a single activation operation, for example by combining multiple "content key and DRM system configuration" data sets into a single data set in a single API call. DASH clients MAY make use of such batching where supported by the platform API. The workflow in this chapter describes the most basic scenario where activation must be performed separately for each content key.

Note: The batching may, for example, be accomplished by concatenating all the `pssh` boxes for the different content keys. Support for this type of batching among DRM systems and platform APIs remains uncommon, despite the potential efficiency gains from reducing the number of license requests triggered.

#### Handling unavailability of content keys #### {#CPS-unavailable-keys}

It is possible that not all of the encrypted [=adaptation sets=] selected for playback can actually be played back (e.g. because a content key for ultra-HD content is only authorized for use on high-security devices). The unavailability of one or more content keys SHOULD NOT be considered a fatal error condition as long as at least one audio and at least one video [=adaptation set=] remains available for playback (assuming both content types are selected for playback to begin with). This logic MAY be overridden by solution specific business logic to better reflect end-user expectations under given conditions.

The set of available content keys can change over time (e.g. due to license expiration or due to new [=periods=] in the presentation requiring different content keys). A DASH client SHALL monitor the set of `default_KID` values that are required for playback and either request the DRM system to make these content keys available or deselect the affected [=adaptation sets=] when the content keys become unavailable. Conceptually, any such change can be handled by re-executing the [[#CPS-selection-workflow|DRM system selection]] and [[#CPS-activation-workflow|activation workflows]], although platform APIs may also offer more fine-grained update capabilities.

A DASH client can request a DRM system to decrypt any set of content keys (if it has the necessary [=DRM system configuration=]). However, this is only a request and it can be denied at multiple stages of processing by different involved entities.

<figure>
	<img src="Diagrams/Security/ReductionOfKeys.png" />
	<figcaption>The set of content keys made available for use can be far smaller than the set requested by a DASH client. Example workflow indicating potential instances of content keys being removed from scope.</figcaption>
</figure>

Common platform media APIs will refuse to start playback if the DRM system does not make available the content keys for all the buffered data. The set of available content keys is only known at the end of executing the DRM system activation workflow and may decrease over time (e.g. due to license expiration).

The proper handling of unavailable keys depends on the limitations imposed by the platform APIs. It may be appropriate for a DASH client to avoid buffering data for encrypted [=adaptation sets=] until the required content key is known to be available. This allows the client to avoid potentially expensive buffer resets and rebuffering if unusable data needs to be removed from buffers.

Note: The DASH client should still download the data into intermediate buffers for faster startup and simply defer submitting it to the platform media API until key availability is confirmed.

If a content key expires during playback, it is common for a media platform to pause playback until the content key can be refreshed with a new license or until data encrypted with the now-unusable content key is removed from buffers. DASH clients SHOULD acquire new licenses in advance of license expiration. Alternatively, DASH clients should implement appropriate recovery/fallback behavior to ensure a minimally disrupted user experience in situations where some content keys remain available.

Issue: EME has no good way to trigger a license request if the key is still available but about to expire, does it? It will only make license requests once the key is no longer available. This makes it hard to proactively refresh licenses.

### Performing license requests ### {#CPS-license-request-workflow}

DASH clients performing license requests SHOULD follow the [[#CPS-lr-model|DASH-IF interoperable license request model]]. The remainder of this chapter only applies to DASH clients that follow this model. Alternative implementations are possible and in common use but are not interoperable and are not described in this document.

DRM systems generally do not perform license requests on their own. Rather, when they determine that a license is required, they generate a document that serves as the license request body and expect the DASH client to deliver it to a license server for processing. The latter returns a suitable response that, if a license is granted, encapsulates the content keys in an encrypted form only readable to the DRM system.

<figure>
	<img src="Diagrams/Security/LicenseRequestConcept.png" />
	<figcaption>Simplified conceptual model of license request processing. Many details omitted.</figcaption>
</figure>

The request and response body are in DRM system specific formats and considered opaque to the DASH client. A DASH client SHALL NOT modify the request body or the response body.

The license request workflow defined here exists to enable the following common business logic needs to be achieved without the need to customize the DASH client with logic specific to a DRM system or license server implementation:

1. Provide proof of authorization if the license server requires the DASH client to prove that the user being served has the rights to use the requested content keys.
1. Execute the license request workflow driven purely by the MPD, without any need for project-specific customization of the DASH client logic.
1. Detect common error scenarios and present an understandable message to the user.

The proof of authorization is optional and the need to attach it to a license request is indicated by the presence of `dashif:authzurl` in the `ContentProtection` descriptor (potentially supplied by business logic instead of the MPD). The proof of authorization is a [[!jwt|JSON Web Token]] in compact encoding (the `aaa.bbb.ccc` form) returned as the HTTP response body when the DASH client performs a GET request to this URL. The token is attached to a license request in the HTTP `Authorization` header with the `Bearer` type.

Error responses from both the authorization service and the license server SHOULD be returned as [[rfc7807]] compatible responses with a 4xx status code and `Content-Type: application/problem+json`.

To process license requests queued during execution of the DRM system activation workflow, the client SHOULD execute the following algorithm:

<div algorithm="drm-license-acquisition">

1. Let <var>pending_license_requests</var> be the set of license requests that the DRM system has requested to be performed, with at least the following data present in each entry:
    * The license request body provided by the DRM system.
    * The `default_KID` associated with the license request.
    * The DRM system configuration data (`ContentProtetion` descriptor) associated with the `default_KID` (as potentially modified/overridden by business logic during DRM system selection workflow).
1. Let <var>pending_authz_requests</var> be a map of `URL -> GUID[]`, with the keys being authorization service URLs and the values being lists of `default_KIDs`. The map is initially empty.
1. For each <var>request</var> in <var>pending_license_requests</var>:
    1. If the DRM system configuration data associated with <var>request</var> does not contain a value for the `dashif:authzurl` element, skip to the next loop iteration.
    1. Create/update the entry in <var>pending_authz_requests</var> with the key being the `dashif:authzurl` value and add the `default_KID` of <var>request</var>.
1. Let <var>authz_tokens</var> be a map of `GUID -> string`, with the keys being `default_KIDs` and the values being the associated authorization tokens. The map is initially empty.
1. For each <var>authz_url</var> and <var>kids</var> pair in <var>pending_authz_requests</var>:
    1. Create a comma-separated list from <var>kids</var>.
    1. Let <var>authz_url_with_kids</var> be <var>authz_url</var> with an additional query string parameter named `kids` with the value from <var>kids</var>.
        * <var>authz_url</var> may already incldue query string parameters, which should be preserved!
    1. Perform an HTTP GET request to <var>authz_url_with_kids</var> (following redirects).
        * Include any relevant session cookies, if present.
        * Custom business logic may provide additional HTTP request headers to enable user identification.
    1. If the response status code indicates failure, make a note of any error information for later processing and skip to the next loop iteration.
        * Authorization services should return `Content-Type: application/problem+json` and an [[rfc7807]] compatible response body in case of failure, potentially enabling user-friendly error processing. However, this is not a strict requirement and other types of errors may also be returned.
    1. For each <var>kid</var> in <var>kids</var>, add an entry to <var>authz_tokens</var> with the key <var>kid</var> and the value being the HTTP response body.
1. For each <var>request</var> in <var>pending_license_requests</var>:
    1. Execute an HTTP POST request with the following parameters:
        * Request body is the license request body from <var>request</var>.
        * Request URL is `dashif:laurl` from <var>request</var>. This license server URL must have been loaded either directly from the MPD or injected by DASH client or custom business logic during DRM system selection.
        * If <var>authz_tokens</var> contains an entry with the key being the `default_KID` from <var>request</var>, add the `Authorization` header with the value being the string `Bearer ` concatenated with the entry value from <var>authz_tokens</var> (e.g. `Bearer aaa.bbb.ccc`).
    1. If the response status code indicates failure, make a note of any error information for later processing and skip to the next loop iteration.
        * License servers should return `Content-Type: application/problem+json` and an [[rfc7807]] compatible response body in case of failure, potentially enabling user-friendly error processing. However, this is not a strict requirement and other types of errors may also be returned.
    1. Supply the HTTP response body to the DRM system for processing.
        * This may cause the DRM system to trigger additional license requests. Append any triggered request to <var>pending_license_requests</var> and copy the attached data (`default_KID` and similar) from the current entry, processing the additional entry in a future iteration of the same loop.
        * If the DRM system indicates a failure to process the data, make a note of any error information for later processing and skip to the next loop iteration.

</div>

Note: While the above algorithm is presented sequentially, authorization requests and license requests may be performed in a parallelized manner to minimize processing time.

At the end of this algorithm, all pending license requests have been performed. However, it is not necessary that all license requests or authorization requests succeed! For example, even if one of the requests needed to obtain an HD quality level content key fails, other requests may still make SD quality level content keys available, leading to a successful playback if the HD quality level is deselected by the DASH client. Individual failing requests therefore do not indicate a fatal error. Rather, such error information should be collected and provided to the top-level error handler of the DRM system activation workflow, which can make use of this data to present user-friendly messages if it decides that meaningful playback cannot take place with the final set of available content keys. See also [[#CPS-unavailable-keys]].

## Additional Constraints for Specific Use Cases ## {#CPS-AdditionalConstraints}

### Efficient license delivery ### {#CPS-efficiency-in-license-requests}

For efficient license delivery, it is recommended that clients:

* Request content keys on the initial processing of an MPD. This is intended to avoid a large number of simultaneous license requests at `MPD@availabilityStartTime`.
* Request content keys for a new [=period=] in advance of its presentation time to allow license download and processing time and prevent interruption of continuous decryption and playback. Advanced requests will also help prevent a large number of simultaneous license requests during a live presentation at `Period@startTime`.

Issue: Describe batching of license requests/responses

### Periodic Re-Authorization ### {#CPS-AdditionalConstraints-PeriodReauth}

This section explains different options and tradeoffs to enable change in content keys (a.k.a. key rotation) on a given piece of content.

Note: The main motivation is to enable access rights changes at program boundaries, not as a measure to increase security of content encryption. The term *Periodic re-authorization* is therefore used here instead of *key rotation*. Note that periodic re-authorization is also one of the ways to implement counting of active streams as this triggers a connection to a license server.

The following use cases are considered:

* Consumption models such as live content, PPV, PVR, VOD, SVOD, live to VOD, network DVR. This includes cases where live content is converted into another consumption model for e.g. catch up TV.
* Regional blackout where client location may be taken into account to deny access to content in a geographical area.

The following requirements are considered:

* Ability to force a client to re-authorize to verify that it is still authorized for content consumption.
* Support seamless and uninterrupted playback when content keys are rotated by preventing storms of license requests from clients (these should be spread out temporally where possible to prevent spiking loads at isolated times), by allowing quick recovery (the system should be resilient if the server or many clients fail), and by providing to the clients visibility into the key rotation signaling.
* Support of hybrid broadcast/unicast networks in which client may operate in broadcast-only mode at least some of the time, e.g. where clients may not  always be able to download licenses on demand through unicast.

This also should not require changes to DASH and the standard processing and validity of MPDs.

#### Periodic Re-Authorization Content Protections Constraints #### {#CPS-AdditionalConstraints-PeriodReauth-Constraints}

Key rotation SHOULD not occur within individual segments. It is usually not necessary to rotate keys within individual segments. This is because segment durations are typically short in live streaming services (on the order of a few seconds), meaning that a segment boundary is usually never too far from the point where key rotation is otherwise desired to take effect.

When key hierarchy is used (see [[#CPS-AdditionalConstraints-PeriodReauth-Implementation]])

* Each Movie Fragment box (`moof`) box SHOULD contain one `pssh` box per DRM system. This `pssh` box SHALL contains sufficient information for obtaining content keys for this fragment when combined with information, for this DRM system, from either the `pssh` box obtained from the Initialization Segment or the `cenc:pssh` element from the MPD and the `KID` value associated with each sample from the `seig` Sample Group Description box and `sbgp` Sample to Group box that lists all the samples that use a given `KID` value.
* All DRM constraints and workflows defined in this document SHALL apply to the EMM/root license (one license is needed per Adaptation Set for each DRM system). Leaf licenses are internal to a DRM system and assumed to be handled by the DRM system directly once the platform media API delivers the `moof/pssh` to the DRM system.

Issue: To be reviewed in light of CMAF and segment/chunk and low latency.

#### Implementation Options #### {#CPS-AdditionalConstraints-PeriodReauth-Implementation}

This section describes recommended approaches for periodic re-authorization. They best cover the use cases and allow interoperable implementation.

Note: Other approaches are possible and may be considered by individual implementers. An example is explicit signaling using e.g. esmg messages, and a custom key rotation signal to indicate future KIDs.

**Period**: A `Period` element is used as the minimum content key duration interval. Content key is rotated at the period boundary. This is a simple implementation and has limitations in the flexibility:

* The existing signaling in the MPD does not allow for early warning of change of the content key and associated decryption context, hence seamless transition between periods is not ensured.
* The logic for the creation of the periods is decided by content creation not DRM systems, hence boundaries may not be suited properly and periods may be longer than the desired key interval.

**Key Hierarchy**: Each DRM system has its own key hierarchy. In general, the number of levels in the key hierarchy varies among DRM systems. For interoperability purposes, only two levels need to be considered:

* Licenses for managing the rights of a user: This can be issued once for enforcing some scope of accessing content, such as a channel or library of shows (existing and future). It is cryptographically bound to one DRM system and is associated with one user ID. It enables access to licenses that control the content keys associated with each show it authorizes. There are many names for this type of licenses. In conditional access systems, a data construct of this type is called an entitlement management message (EMM). In the PlayReady DRM system, a license of this type is called a “root license”. There is no agreement on a common terminology.
* Licenses for accessing the content: This is a license that contains content keys and can only be accessed by devices that have been authorized. While licenses for managing rights are most of the time unique per user, the licenses for accessing the content are not expected to be unique and are tied to the content and not a user, therefore these may be delivered with content in a broadcast distribution. In addition doing so allows real time license acquisition, and do not require repeating client authentication, authorization, and rebuilding the security context with each content key change in order to enable continuous playback without interruption cause be key acquisition or license processing. In conditional access systems, a data construct of this type is called an entitlement control message (ECM). In the PlayReady DRM system,  a license of this type is called a “leaf license”. There is no agreement on a common terminology.

When using key hierarchy, the `@cenc:default_KID` value in the `ContentProtection` element, which is also in the `tenc` box, is the ID of the key requested by the DRM client. These keys are delivered as part of acquisition of the rights for a user. The use of key hierarchy is optional and DRM system specific.

Issue: For key hierarchy, add a sentence explaining that mixing DRM systems is possible with system constraints.

### Low Latency ### {#CPS-AdditionalConstraints-LowLatency}

Low latency content delivery requires that all components of the end-to-end systems are optimized for reaching that goal. DRM systems and the mechanisms used for granting access also need to be used in specific manners to minimize the impact on the latency. DRM systems are involved in the access to content in several manners:

* Device initialization
* Device authorization
* Content access granting

Each of these steps can have from an impact on latency ranging from low to high. The following describes possible optimizations for minimizing the latency.

#### Licenses Pre-Delivery #### {#CPS-AdditionalConstraints-LowLatency-Predelivery}

In a standard playback session, a client, after receiving the DASH MPD, checks the `@cenc:default_KID` value (either part of the `mp4protection` element or part of a DRM system element). If the client already has a content key associated to this `KID` value, it can safely assume that it is able to get access to content. If it does not have such content key, then a license request is triggered. This process is done every time a MPD is received (change of `Period`, change of Live service, notification of MPD change …). It would therefore be better that the client always has all keys associated to `@cenc:default_KID` values. One mechanism is license pre-delivery. Predelivery can be performed in different occasions:

* When launching the application, the client needs to perform some initialization and refresh of data, it therefore connects to many servers for getting updated information. The license server SHOULD allow the client to receive licenses for accessing content the client is entitled to. Typically, for subscription services, all licenses for all Live services SHOULD be delivered during this initialization step. It is the DRM system client responsibility to properly store the received information.
* The DRM system SHOULD have a notification mechanism allowing to trigger a client or a set of clients to out-of-band initiate licenses request, so that it is possible to perform license updates in advance. This typically allows pre-delivery of licenses when a change will occur at a `Period` boundary and, in this case, this also allow avoiding all clients connecting at almost the same time to the license server if not triggered in advance randomly.
* In case a device needs nevertheless to retrieve a license, the DRM system MAY also batch responses into a single transaction allowing to provide additional licenses (as explained in Section [[#CPS-activation-workflow]]) that can be used in the future.

#### Key Hierarchy and CMAF Chunked Content #### {#CPS-AdditionalConstraints-Low-chunkedContent}

When a DRM system uses key hierarchy for protecting content, it adds DRM information in both possibly the Initialization Segment and in the content (in the `moof` box). The information in the `moof` box can allow the DRM client to know which root key to use decrypt the leaf license or to identify the already decrypted content key from a local protected storage. Most of the processing and logic is DRM system-specific and involves DRM system defined encryption and signaling. It may also include additional steps such as evaluating leaf license usage rules. Key hierarchy is one technique for enabling key rotation and it is not required to rotate content key at high frequency, typically broadcast TV has content key cryptoperiods of 10 seconds to few minutes.

CMAF chunked Content introduces `moof` boxes at a high frequency as it appears within segments and not only at the beginning of a segment. One can therefore expect to have several `moof` boxes every second. Adding signaling SHOULD be done only in the `moof` box of the first chunk in a segment.

Issue: To be completed. Look at encryption: Key available for license server “early” for been able to generate licenses (root or leaf licenses). Avoid the license server been on the critical path. Encourage license persistence in the client.

### Use of W3C Clear Key with DASH ### {#CPS-AdditionalConstraints-W3C}

When using W3C Clear Key key system with DASH [[!encrypted-media]], Clear Key related signaling is included in the MPD with a `ContentProtection` element that has the following format.

The Clear Key `ContentProtection` element attributes SHALL take the following values:

* The UUID e2719d58-a985-b3c9-781a-b030af78d30e is used for the `@schemeIdUri` attribute.
* The `@value` attribute is equal to the string “ClearKey1.0”

The following element MAY be added under the `ContentProtection` element:

* A `Laurl` element that contains the URL for a Clear Key license server allowing to receive a Clear Key license in the format defined in [[!encrypted-media]] section 9.1.4. It has the attribute `@Lic_type` that is a string describing the license type served by this license server. Possible value is “EME-1.0” when the license served by the Clear Key license server is in the format defined in [[!encrypted-media]] section 9.1.4.

The name space for the `Laurl` element is `http://dashif.org/guidelines/clearKey`

An example of a Clear Key `ContentProtection` element is as follows

```xml
<xs:schema xmlns:ck=http://dashif.org/guidelines/clearKey>
<ContentProtection
  schemeIdUri="urn:uuid:1077efec-c0b2-4d02-ace3-3c1e52e2fb4b"
  value="ClearKey1.0">
  <ck:Laurl Lic_type="EME-1.0">
    https://clearKeyServer.foocompany.com
  </ck:Laurl>
</ContentProtection>
```

W3C also specifies the use of the DRM systemID=”1077efec-c0b2-4d02-ace3-3c1e52e2fb4b” in [[!eme-initdata-cenc]] section 4 to indicate that tracks are encrypted with Common Encryption [[!MPEGCENC]], and list the `KID` of content keys used to encrypt the track in a version 1 `pssh` box with that DRM systemID.  However, the presence of this Common `pssh` box does not indicate whether content keys are managed by DRM systems or Clear Key management specified in this section. Browsers are expected to provide decryption in the case where Clear Key management is used, and a DRM system where a DRM key management system is used. Therefore, clients SHALL NOT rely on the signalling of DRM systemID 1077efec-c0b2-4d02-ace3-3c1e52e2fb4b as an indication that the Clear Key mechanism is to be used.

W3C specifies that in order to activate the Clear Key mechanism, the client must provide Clear Key initialization data to the browser. The Clear Key initialization data consists of a listing of the default KIDs required to decrypt the content.

The MPD SHOULD NOT contain Clear Key initialization data. Instead, clients SHALL construct Clear Key initialization data at runtime, based on the default KIDs signaled in the MPD using `ContentProtection` elements with the urn:mpeg:dash:mp4protection:2011 scheme.

When requesting a Clear Key license to the license server, it is recommended to use a secure connection as described in Section [[#CPS-HTTPS]].

When used with a license type equal to “EME-1.0”:

* The GET request for the license includes in the body the JSON license request format defined in [[!encrypted-media]] section 9.1.3. The license request MAY also include additional authentication elements such as access token, device or user ID.
* The response from the license server includes in the body the Clear Key license in the format defined in [[!encrypted-media]] section 9.1.4 if the device is entitled to receive the Content Keys.

It should be noted that clients receiving content keys through the Clear Key key system may not have the same robustness that typical DRM clients are required to have. When the same content keys are distributed to DRM clients and to weakly-protected or unprotected clients, the weakly-protected or unprotected clients become a weak link in the system and limits the security of the overall system.
