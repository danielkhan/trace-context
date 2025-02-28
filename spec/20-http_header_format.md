# Trace Context HTTP Headers Format

This section describes the binding of the distributed trace context to `traceparent` and `tracestate` HTTP headers.

## Relationship Between the Headers

The `traceparent` header represents the incoming request in a tracing system in a common format, understood by all vendors. Here’s an example of a `traceparent `header.

``` http
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```

The `tracestate` header includes the parent in a potentially vendor-specific format:

``` http
tracestate: congo=t61rcWkgMzE
```

For example, say a client and server in a system use different tracing vendors: Congo and Rojo. A client traced in the Congo system adds the following headers to an outbound HTTP request.

``` http
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
tracestate: congo=t61rcWkgMzE
```

**Note**: In this case, the `tracestate `value `t61rcWkgMzE` is the result of Base64 encoding the parent ID (`b7ad6b7169203331`), though such manipulations are not required.

The receiving server, traced in the Rojo tracing system, carries over the `tracestate` it received and adds a new entry to the left.

``` http
traceparent: 00-0af7651916cd43dd8448eb211c80319c-00f067aa0ba902b7-01
tracestate: rojo=00f067aa0ba902b7,congo=t61rcWkgMzE
```

You'll notice that the Rojo system reuses the value of it’s `traceparent` for its entry in `tracestate`. This means it is a generic tracing system (no proprietary information is being passed). Otherwise, `tracestate` entries are opaque and can be vendor-specific.

If the next receiving server uses Congo, it carries over the `tracestate` from Rojo and adds a new entry for the parent to the left of the previous entry.

``` http
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b9c7c989f97918e1-01
tracestate: congo=ucfJifl5GOE,rojo=00f067aa0ba902b7
```

**Note:** `ucfJifl5GOE` is the Base64 encoded parent ID `b9c7c989f97918e1`.

Notice when Congo wrote its `traceparent` entry, it is not encoded, which helps in consistency for those doing correlation. However, the value of its entry `tracestate` is encoded and different from `traceparent`. This is ok.

Finally, you'll see `tracestate` retains an entry for Rojo exactly as it was, except pushed to the right. The left-most position lets the next server know which tracing system corresponds with `traceparent`. In this case, since Congo wrote `traceparent`, its `tracestate` entry should be left-most.


## Traceparent Header

The `traceparent` HTTP header field identifies the incoming request in a tracing system. It has four fields:

* version
* trace-id
* parent-id
* trace-flags


### Header Name

Header name: `traceparent`

In order to increase interoperability across multiple protocols and encourage successful integration, by default vendors SHOULD keep the header name lowercase. The header name is a single word without any delimiters, for example, a hyphen (`-`).

Vendors MUST expect the header name in any case (upper, lower, mixed), and SHOULD send the header name in lowercase.

### traceparent Header Field Values

This section uses the Augmented Backus-Naur Form (ABNF) notation of [[RFC5234](https://w3c.github.io/trace-context/#bib-rfc5234)], including the DIGIT rule from that document. `DIGIT` rule defines a single number character `0`-`9`.

``` abnf
HEXDIGLC = DIGIT / "a" / "b" / "c" / "d" / "e" / "f" ; lowercase hex character
value           = version "-" version-format
```

The dash (`-`) character is used as a delimiter between fields.

#### version

``` abnf
version         = 2HEXDIGLC   ; this document assumes version 00. Version 255 is forbidden
```

The value is US-ASCII encoded (which is UTF-8 compliant).

Version (`version`) is a 1 byte representing an 8-bit unsigned integer. Version `255` is invalid. The current specification assumes the `version` is set to `00`.

#### version-format

The following `version-format` definition is used for version `00`.

``` abnf
version-format   = trace-id "-" parent-id "-" trace-flags
trace-id         = 32HEXDIGLC  ; 16 bytes array identifier. All zeroes forbidden
parent-id        = 16HEXDIGLC  ; 8 bytes array identifier. All zeroes forbidden
trace-flags      = 2HEXDIGLC   ; 8 bit flags. Currently, only one bit is used. See below for details
```

#### trace-id

This is the ID of the whole trace forest and is used to uniquely identify a [distributed trace](https://w3c.github.io/trace-context/#dfn-distributed-traces) through a system. It is represented as a 16-byte array, for example, `4bf92f3577b34da6a3ce929d0e0e4736`. All bytes as zero (`00000000000000000000000000000000`) is considered an invalid value.


A vendor SHOULD generate globally unique values for `trace-id`. Many unique identification generation algorithms create IDs where one part of the value is constant (often time- or host-based), and the other part is a randomly generated value. Because tracing systems may make sampling decisions based on the value of `trace-id`, for increased interoperability vendors MUST  keep the random part of `trace-id` ID on the left side.


When a system operates with a `trace-id` that is shorter than 16 bytes, it SHOULD fill-in the extra bytes with random values rather than zeroes. Let's say the system works with an 8-byte `trace-id` like `3ce929d0e0e4736`. Instead of setting `trace-id` value to `0000000000000003ce929d0e0e4736` it SHOULD generate a value like `4bf92f3577b34da6a3ce929d0e0e4736` where`4bf92f3577b34da6a` is a random value or a function of time and host value.


**Note**: Even though a system may operate with a shorter `trace-id` for [distributed trace](https://w3c.github.io/trace-context/#dfn-distributed-traces) reporting, the full `trace-id` MUST be propagated to conform to the specification.

If the `trace-id` value is invalid (for example if it contains non-allowed characters or all zeros), vendors MUST ignore the `traceparent`.

##### parent-id

This is the ID of this request as known by the caller (in some tracing systems, this is known as the `span-id`, where a `span` is the execution of a client request). It is represented as an 8-byte array, for example, `00f067aa0ba902b7`. All bytes as zero (`0000000000000000`) is considered an invalid value.

Vendors MUST ignore the `traceparent` when the `parent-id` is invalid (for example, if it contains non-lowercase hex characters).

##### trace-flags

An <a data-cite='!BIT-FIELD'>8-bit field</a>  that controls tracing flags such as sampling, trace level, etc. These flags are recommendations given by the caller rather than strict rules to follow for three reasons:

1. Trust and abuse
2. Bug in the caller
3. Different load between caller service and callee service might force callee to downsample.

You can find more in the section [Security considerations](#security-considerations) of this specification.

Like other fields, `trace-flags` is hex-encoded. For example, all `8` flags set would be `ff` and no flags set would be `00`.

As this is a bit field, you cannot interpret flags by decoding the hex value and looking at the resulting number. For example, a flag `00000001` could be encoded as `01` in hex, or `09` in hex if present with the flag `00001000`. A common mistake in bit fields is forgetting to mask when interpreting flags.

Here is an example of properly handling trace flags:

``` java
static final byte FLAG_SAMPLED = 1; // 00000001
...
boolean sampled = (traceFlags & FLAG_SAMPLED) == FLAG_SAMPLED;
```

The current version of this specification (`00`) only supports a single flag called `sampled`.

When set, the least significant bit (right-most), denotes that the caller may have recorded trace data. When unset, the caller did not record trace data out-of-band.

There are a number of recording scenarios that may break distributed tracing:

- Only recording a subset of requests results in broken traces.
- Recording information about all incoming and outgoing requests becomes prohibitively expensive, at load.
- Making random or component-specific data collection decisions leads to fragmented data in all traces.

Because of these issues, tracing vendors make their own recording decisions, and there is no consensus on what is the best algorithm for this job.

Various techniques include:

- Probability sampling (sample 1 out of 100 <a>distributed traces</a> by flipping a coin)
- Delayed decision (make collection decision based on duration or a result of a request)
- Deferred sampling (let callee decide whether information about this request need to be collected)

How these techniques are implemented can be tracing vendor-specific or application-defined.

The `tracestate` field is designed to handle the variety of techniques for making recording decisions (or other specific information) specific for a given vendor. The `sampled` flag provides better interoperability between vendors. It allows vendors to communicate recording decisions and enable a better experience for the customer.

For example, when a SaaS service participates in a <a>distributed trace</a>, this service has no knowledge of the tracing vendor used by its caller. This service may produce records of incoming requests for monitoring or troubleshooting purposes. The `sampled` flag can be used to ensure that information about requests that were marked for recording by the caller will also be recorded by SaaS service downstream so that the caller can troubleshoot the behavior of every recorded request.

The `sampled` flag has no restriction on its mutations except that it can only be mutated when [parent-id is updated](#parent-id).

The following are a set of suggestions that vendors SHOULD use to increase vendor interoperability.

- If a component made definitive recording decision - this decision SHOULD be reflected in the `sampled` flag.
- If a component needs to make a recording decision - it SHOULD respect the `sampled` flag value.
  [Security considerations](#security-considerations) SHOULD be applied to protect from abusive or malicious use of this flag.
- If a component deferred or delayed the decision and only a subset of telemetry will be recorded, the `sampled` flag should be propagated unchanged. It should be set to `0` as the default option when the trace is initiated by this component.

There are two additional options that vendors MAY follow:

- A component that makes a deferred or delayed recording decision may communicate the priority of a recording by setting `sampled` flag to `1` for a subset of requests.
- A component may also fall back to probability sampling and set the `sampled` flag to `1` for the subset of requests.

##### Other Flags

The behavior of other flags, such as (`00000100`) is not defined and is reserved for future use. Vendors MUST set those to zero.

### Examples of HTTP traceparent Headers

*Valid traceparent when caller sampled this request:*

```
Value = 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
base16(version) = 00
base16(trace-id) = 4bf92f3577b34da6a3ce929d0e0e4736
base16(parent-id) = 00f067aa0ba902b7
base16(trace-flags) = 01  // sampled
```

*Valid traceparent when caller didn’t sample this request:*

```
Value = 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-00
base16(version) = 00
base16(trace-id) = 4bf92f3577b34da6a3ce929d0e0e4736
base16(parent-id) = 00f067aa0ba902b7
base16(trace-flags) = 00  // not sampled
```

### Versioning of traceparent

This specification is opinionated about future version of the trace context. The current version of this specification assumes that future versions of `traceparent` header will be additive to the current one.

Vendors MUST follow these rules when parsing headers with an unexpected format:

- Pass-through services should not analyze the version. They should expect that headers may have larger size limits in the future and only disallow prohibitively large headers.
- When the version prefix cannot be parsed (it's not 2 hex characters followed by a dash (`-`), the implementation should restart the trace.
- If a higher version is detected, the implementation SHOULD try to parse it by trying the following:
    - If the size of the header is shorter than 55 characters, the vendor should not parse the header and should restart the trace.
    - Parse `trace-id` (from the first dash through the next 32 characters). Vendors MUST check that the 32 characters are hex, and that they are followed by a dash (`-`).
    - Parse `parent-id` (from the second dash at the 35th position through the next 16 characters). Vendors MUST check that the 16 characters are hex and followed by a dash.
    - Parse the `sampled` bit of `flags` (2 characters from the third dash). Vendors MUST check that the 2 characters are either the end of the string or a dash.

    If all three values were parsed successfully, the vendor should use them.


Vendors MUST NOT parse or assume anything about unknown fields for this version. Vendors MUST use these fields to construct the new `traceparent` field according to the highest version of the specification known to the implementation (in this specification it is `00`).

## Tracestate Header

The main purpose of the `tracestate` HTTP header is to provide additional vendor-specific trace identification information across different distributed tracing systems and is a companion header for the `traceparent` field. It also conveys information about the request’s position in multiple distributed tracing graphs.

If the vendor failed to parse `traceparent`, it MUST NOT attempt to parse `tracestate`. Note that the opposite is not true: failure to parse `tracestate` MUST NOT affect the parsing of `traceparent`.

### Header Name

Header name: `tracestate`

In order to increase interoperability across multiple protocols and encourage successful integration, by default you SHOULD keep the header name lowercase. The header name is a single word without any delimiters, for example, a hyphen (`-`).

Vendors MUST expect the header name in any case (upper, lower, mixed), and SHOULD send the header name in lowercase.

##### tracestate Header Field Values

The `tracestate` field may contain any opaque value in any of the keys. Multiple `tracestate` headers are allowed. Values from multiple headers in incoming requests SHOULD be combined in a single header according to [Field Order](https://httpwg.org/specs/rfc7230.html#page-24) [[RFC7230](https://w3c.github.io/trace-context/#bib-rfc7230)], and sent as a single header in an outgoing request.

This section uses the Augmented Backus-Naur Form (ABNF) notation of [[RFC5234](https://w3c.github.io/trace-context/#bib-rfc5234)], including the DIGIT rule in [appendix B.1 for RFC5234](https://tools.ietf.org/html/rfc5234#appendix-B.1). It also includes the `OWS` rule from [RFC7230 section 3.2.3](https://httpwg.org/specs/rfc7230.html#section-3.2.3).

`DIGIT` rule defines number `0`-`9`.

The `OWS` rule defines an optional whitespace character. To improve readability, it is used where zero or more whitespace characters might appear.

The caller SHOULD generate the optional whitespace as a single space; otherwise, a caller SHOULD NOT generate optional whitespace. See details in the [corresponding RFC](https://httpwg.org/specs/rfc7230.html#section-3.2.3).

The `tracestate` field value is a `list` of `list-members` separated by commas (`,`). A `list-member` is a key/value pair separated by an equals sign (`=`). Spaces and horizontal tabs surrounding `list-member`s are ignored. There can be a maximum of 32 `list-member`s in a `list`.

Empty and whitespace-only list members are allowed. Vendors MUST accept empty `tracestate `headers but SHOULD avoid sending them. Empty list members are allowed in `tracestate` because it is difficult for a vendor to recognize the empty value when multiple `tracestate` headers are sent. Whitespace characters are allowed for a similar reason, as some vendors automatically inject whitespace after a comma separator, even in the case of an empty header.

#### list

A simple example of a `list` with two `list-member`s might look like:
`vendorname1=opaqueValue1,vendorname2=opaqueValue2`.

``` abnf
list  = list-member 0*31( OWS "," OWS list-member )
list-member = key "=" value
list-member = OWS
```

Identifiers for a `list` are short (up to 256 characters) textual identifiers.

#### list-members

A` list-member` contains a key/value pair.

##### Key

The key is an identifier that describes the vendor.

``` abnf
key = lcalpha 0*255( lcalpha / DIGIT / "_" / "-"/ "*" / "/" )
key = lcalpha 0*240( lcalpha / DIGIT / "_" / "-"/ "*" / "/" ) "@" lcalpha 0*13( lcalpha / DIGIT / "_" / "-"/ "*" / "/" )
lcalpha    = %x61-7A ; a-z
```

**Note**: Identifiers MUST begin with a lowercase letter, and can only contain lowercase letters (`a`-`z`), digits (`0`-`9`), underscores (`_`), dashes (`-`), asterisks (`*`), and forward slashes (`/`).

For multi-tenant vendors scenarios, an at sign (`@`) sign can be used to prefix the vendor name. Vendors SHOULD set the tenant ID at the beginning of the key. For example, \
`fw529a3039@dt` - `fw529a3039` is a tenant ID and `@dt` is a vendor name. Searching for `@dt=` is more robust for parsing (for example, searching for all a vendor's keys).

##### Value

The value is opaque string up to 256 characters printable ASCII [[RFC0020](https://w3c.github.io/trace-context/#bib-rfc0020)] characters (i.e., the range 0x20 to 0x7E) except comma (`,`) and (`=`). Note that this also excludes tabs, newlines, carriage returns, etc.

``` abnf
value    = 0*255(chr) nblk-chr
nblk-chr = %x21-2B / %x2D-3C / %x3E-7E
chr      = %x20 / nblk-chr
```

#### Combined Header Value

The `tracestate` value is the concatenation of trace graph key/value pairs

Example: `vendorname1=opaqueValue1,vendorname2=opaqueValue2`

Only one entry per key is allowed because the entry represents that last position in the trace. Hence vendors must overwrite their entry upon reentry to their tracing system.

For example, if a vendor name is Congo and a trace started in their system and then went through a system named Rojo and later returned to Congo, the `tracestate` value would not be:

`congo=congosFirstPosition,rojo=rojosFirstPosition,congo=congosSecondPosition`

Instead, the entry would be rewritten to only include the most recent position:
`congo=congosSecondPosition,rojo=rojosFirstPosition`

#### tracestate Limits:

The `tracestate` field contains essential information for request correlation. Vendors MUST propagate this field. There might be multiple `tracestate` headers in a single request according to [RFC7230 section 3.2.2](https://tools.ietf.org/html/rfc7230#section-3.2.2). Vendors may propagate them as they came, combine them into a single header, or split them into multiple headers differently, following the RFC specification.

Vendors SHOULD propagate at least 512 characters of a combined header. This length includes commas required to separate list items and optional white space (`OWS`) characters.

There are systems where propagating of 512 characters of `tracestate` may be expensive. In this case, the maximum size of the propagated `tracestate` header SHOULD be documented and explained. The cost of propagating `tracestate` SHOULD be weighted against the value of monitoring scenarios enabled for the end users.

In a situation where `tracestate` needs to be truncated due to size limitations, the vendor MUST truncate whole entries. Entries larger than `128` characters long SHOULD be removed first. Then entries SHOULD be removed starting from the end of `tracestate`. Note that other truncation strategies like safe list entries, blocked list entries, or size-based truncation MAY be used, but are highly discouraged. Those strategies decrease the interoperability of various tracing vendors.

### Examples of tracestate HTTP Headers

Single tracing system (generic format):

``` http
tracestate: rojo=00f067aa0ba902b7
```

Multiple tracing systems (with different formatting):

``` http
tracestate: rojo=00f067aa0ba902b7,congo=t61rcWkgMzE
```

### Versioning of tracestate

The version of `tracestate` is defined by the version prefix of `traceparent` header. Vendors need to attempt to parse `tracestate` if a higher version is detected, to the best of its ability. It is the vendor’s decision whether to use partially-parsed `tracestate` key/value pairs or not.


## Mutating the traceparent Field

A vendor receiving a `traceparent` request header MUST send it to outgoing requests. It MAY mutate the value of this header before passing it to outgoing requests.

If the value of the `traceparent` field wasn't changed before propagation, `tracestate` MUST NOT be modified as well. Unmodified headers propagation is typically implemented in pass-through services like proxies. This behavior may also be implemented in a service which currently does not collect distributed tracing information.

Following is the list of allowed mutations:

- **Update `parent-id`**: The value of [the  parent-id field](#parent-id) can be set to the new value representing the ID of the current operation. This is the most typical mutation and should be considered a default.
- **Update `sampled`**: The value of [the sampled field](#trace-flags) reflects the caller's recording behavior: either trace data was dropped or may have been recorded out-of-band. This can be indicated by toggling the flag in both directions. This mutation gives the downstream vendor information about the likelihood that its parent's information was recorded. The `parent-id` field MUST be set to a new value with the `sampled` flag update.
- **Restart trace**: All properties (`trace-id`, `parent-id`, `trace-flags`) are regenerated. This mutation is used in services that are defined as a front gate into secure networks and eliminates a potential denial-of-service attack surface. Vendors SHOULD clean up `tracestate` collection on `traceparent` restart. There are rare cases when the original `tracestate` entries must be preserved after a restart. This typically happens when the `trace-id` is reverted back at some point of the trace flow, for instance, when it leaves the secure network. However, it SHOULD be an explicit decision, and not the default behavior.
- **Downgrade the version**: This version of the specification (`00`) [defines the behavior](#version) for a vendor that receives a `traceparent` header of a higher version. In this case, the first mutation is to downgrade the version of the header. Other mutations are allowed in combination with this one.

Vendors MUST NOT make any other mutations to the `traceparent` header.

## Mutating the tracestate Field

Vendors receiving a `tracestate` request header MUST send it to outgoing requests. It MAY mutate the value of this header before passing to outgoing requests. When mutating `tracestate`, the order of unmodified key/value pairs MUST be preserved. Modified keys MUST be moved to the beginning (left) of the list.

Following are allowed mutations:

- **Update key value**. The value of any key can be updated. Modified keys MUST be moved to the beginning (left) of the list. This is the most common mutation resuming the trace.
- **Add a new key/value pair**. The new key-value pair SHOULD be added to the beginning of the list.
- **Delete a key/value pair**. Any key/value pair MAY be deleted. Vendors SHOULD NOT delete keys that were not generated by them. The deletion of an unknown key/value pair will break correlation in other systems. This mutation enables two scenarios. The first is that proxies can block certain `tracestate` keys for privacy and security concerns. The second scenario is a truncation of long `tracestate`'s.
