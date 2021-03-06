# Http Request Correlation Specification

In order to track a complex operation in a multi-tiered system, it is important to track the relationships between individual actions in this operation. For systems communicating over http protocol, each request needs to carry a context that can be used to build up this relationship.

If requests are sent via HTTP, it is necessary to pass this context along with the HTTP packet itself. In systems where a single vendor controls communication on both sides of the HTTP pipeline, it is relatively simple to define some vendor-specific scheme, which defines a request context and passes it in the HTTP headers. Indeed today there are a number of such schemes, but they do not interoperate with each other.

There are environments where multi-tiered applications use components from different vendors. Some of those components may be out of the user control due to organizational silos. Some be part of a public cloud infrastructure and controlled by cloud provider. Some may have pre-baked support for distributed traces that cannot be altered. It is useful to have uniform standard for the distributed context semantic and a format that would give each vendor-specific logging system the information it needs to trace the relationships among requests for the system as a whole. 

Most of the vendors define two pieces of request context - unique identifier of the request and properties defining the overall complex operation (distributed trace). The [OpenTracing](http://opentracing.io/documentation/) Baggage concept [spec](https://github.com/opentracing/specification/blob/master/specification.md)) needs exactly this feature.   This mechanism can be used to propagate logging control information for more advanced logging features, and would likewise benefit from standardization.  

Ideally such a standard
1. Provides enough structure so that each vendor-specific logging system can get the information it needs from the parts of the system that are not
under the vendor's control (but conform to the standard)

2. Provides enough flexibility so vendors can easily modify their systems to conform to the standard and each vendor can innovate and provide advanced features within its logging system.

This document describes such a standard, and the rational for the design choices (which should be based in the two principles above). 

# Http Header Fields

Http has the concept of a [Header Fields](https://tools.ietf.org/html/rfc7230#section-3.2)
for passing addition information with the request. This standard suggests defining  
the following new header fields:

* `Request-Id`: A string representing the unique identifier for this request.
* `Correlation-Context`:  A list of comma separated set of strings of the form KEY=VALUE.
These values are intended for control of logging systems and should be passed along to any child requests.  

The exact syntax for the values of both of these fields is given in its own section below.

## The Request-Id Field

Section [The Expected Environment for Using Request-Id](#The-Expected-Environment-for-Using-Request-Id) explains motivation of this format.

This protocol defines a strict `Request-Id` field format and a set of fall back rules that allow to experiment and move the protocol forward while keeping systems following the protocol with the required information.

`Request-Id` should follow the following format:

```
<trace-id>[-<span-base-id>[.<level>.<level>]]
```

Where:
- `<trace-id>` - identifies overall distributed trace. 22 base64 characters (* 6 = 132 bits ~= 16 bytes).
- `<span-base-id>` - Optional: defines the span that is base for the hierarchy of other spans. Not more than 11 base64 characters.
- `<level>` - Optional: sequence number of a call made by specific layer. Not more than 10 base10 characters (32 bit unsigned).
- maximum length of the header value should be 128 bytes.

See section [Base64 encoding of binary blobs](#Base64-encoding-of-binary-blobs) for the details of base64 encoding.

There are three types of operations that can be made with the `Request-Id`:
- **Extend**: used to create a new unique request ID. Implementation of this operation MUST append `.0` to the request. Since the request ID should be unique - it is recommended to use **Reset** operation instead of **Extend** for the untrusted caller, which can send the same ID multiple times.

    *Third layer:*

    Request-Id = `3qdi2JDFioDFjDSF223f23-SdfD8DF908D.1.4`
    Extend(Request-Id) = `3qdi2JDFioDFjDSF223f23-SdfD8DF908D.1.4.0`.

    *First layer after Reset*

    Request-Id = `3qdi2JDFioDFjDSF223f23-SdfD8DF908D`
    Extend(Request-Id) = `3qdi2JDFioDFjDSF223f23-SdfD8DF908D.0`.

    *First layer for the plan <trace-id>*

    Request-Id = `3qdi2JDFioDFjDSF223f23`
    Extend(Request-Id) = `3qdi2JDFioDFjDSF223f23.0`.

- **Increment**: used to mark the "next" call to the dependant service. Increments that number after the last `.` in base10.
    Request-Id = `3qdi2JDFioDFjDSF223f23-SdfD8DF908D.3`
    Increment(Request-Id) = `3qdi2JDFioDFjDSF223f23-SdfD8DF908D.4`

- **Reset**: preserves `<trace-id>` and generate a new 11 base64 characters `<span-base-id>` without any hierarchy. Used to reset the long hierarchical string or as a replacement for either **Extend** or **Increment**. 
    Request-Id = `3qdi2JDFioDFjDSF223f23-SdfD8DF908D`
    Reset(Request-Id) = `3qdi2JDFioDFjDSF223f23-MGY3gOT5kgZ`

This protocol expects every actor in a system to modify the `Request-Id` using one of the actions above. There are three scenarios how this protocol can be used.

### Scenario 1. Reset all the time

Resetting of request is the most straightforward operation. Reset can also benefit from existing request identifiers that server like nginx may already have.

```
Client sends: 3qdi2JDFioDFjDSF223f23
    A logs request: Reset(3qdi2JDFioDFjDSF223f23) => 3qdi2JDFioDFjDSF223f23-MGY3gOT5kgZ 
                    with the parent 3qdi2JDFioDFjDSF223f23

    A sends request to B: Reset(3qdi2JDFioDFjDSF223f23-MGY3gOT5kgZ) => 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm
                            and logs the call with the request ID 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm
        
        B logs request: Reset(3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm) => 3qdi2JDFioDFjDSF223f23-MoeykjJSsoJ 
                        with the parent 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm

    A sends request to C: Reset(3qdi2JDFioDFjDSF223f23-MGY3gOT5kgZ) => 3qdi2JDFioDFjDSF223f23-F1YWhhcm5mM
                            and logs the call with the request ID 3qdi2JDFioDFjDSF223f23-F1YWhhcm5mM
        
        C logs request: Reset(3qdi2JDFioDFjDSF223f23-F1YWhhcm5mM) => 3qdi2JDFioDFjDSF223f23-OTh5NHVoZG5 
                        with the parent 3qdi2JDFioDFjDSF223f23-F1YWhhcm5mM
```

### Scenario 2. Extend + Increment all the time

Extend and Increment are useful for lossy telemetry systems. With only few requests 

```
Client sends request to A: 3qdi2JDFioDFjDSF223f23
    A logs request: Extend(3qdi2JDFioDFjDSF223f23) => 3qdi2JDFioDFjDSF223f23.0
                    with the parent 3qdi2JDFioDFjDSF223f23

    A sends request to B: Increment(3qdi2JDFioDFjDSF223f23.0) => 3qdi2JDFioDFjDSF223f23.1
                            and logs the call with the request ID 3qdi2JDFioDFjDSF223f23.1
        
        B logs request: Extend(3qdi2JDFioDFjDSF223f23.1) => 3qdi2JDFioDFjDSF223f23.1.0
                        with the parent 3qdi2JDFioDFjDSF223f23.0

    A sends request to C: Increment(3qdi2JDFioDFjDSF223f23.1) => 3qdi2JDFioDFjDSF223f23.2
                            and logs the call with the request ID 3qdi2JDFioDFjDSF223f23.1
        
        C logs request: Extend(3qdi2JDFioDFjDSF223f23.2) => 3qdi2JDFioDFjDSF223f23.2.0
                        with the parent 3qdi2JDFioDFjDSF223f23.2
```

### Scenario 3. Mixed scenario

Load balancers and proxies may be a complete black box for the tracing system. So it may be useful to preserve the correlation for the trace went through it. Especially for the multiple retries scenarios. In the example below, you can correlate the request sent from A `3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm` with the request logged by B `3qdi2JDFioDFjDSF223f23-M2QgMWEgYjA` as its parent is `3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm.1`.

```
Client sends request to A: 3qdi2JDFioDFjDSF223f23
    A logs request: Reset(3qdi2JDFioDFjDSF223f23) => 3qdi2JDFioDFjDSF223f23-MGY+gOT5kgZ 
                    with the parent 3qdi2JDFioDFjDSF223f23

    A sends request to B: Reset(3qdi2JDFioDFjDSF223f23-MGY+gOT5kgZ) => 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm
                            and logs the call with the request ID 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm
        
        B logs request: Extend(3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm) => 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm.0
                        with the parent 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm

        B sends request to C: Increment(3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm.0) => 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm.1
                                and logs the call with the request ID 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm.1
        
                C logs request: Reset(3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm.1) => 3qdi2JDFioDFjDSF223f23-M2QgMWEgYjA
                        with the parent 3qdi2JDFioDFjDSF223f23-5NHVoZG5NTm.1
```

### Fallback options

When the format of `Request-Id` does not match the expected format, the following fallback options should be applied:

#### `<trace-id>` format mismatch

Any string with the allowed characters up to the first `-`, `.` or to the very end of the string should be treated as a `<trace-id>`. Depending on vendors limitation protocol defines four behaviors in this priority order:

1. Use `<trace-id>` to log trace and propagate further even if do not match the expected format.
2. Use derived `<trace-id>` (hashed value) to log trace and propagate the **original** value further. This option may be applied if vendor's telemetry storage expects `<trace-id>` as a long number or when the `<trace-id>` string is too long to store.
3. Use derived `<trace-id>` (hashed value) to log trace and propagate the hashed value further. This option may be applied when propagation of `<trace-id>` between incoming and outgoing request processing is not possible in a form it was received.
4. Restart the trace with the fresh `Request-Id` that matches the format. This option can be applied when neither storing nor propagation possible with the `<trace-id>` in a form it was received. It also can be used for untrusted caller system.

For hashing - use algorithm described in [Hashing for Fixed Sized ID Systems](#Hashing for-Fixed-Sized-ID-Systems) to hash.
When recording hashed value - consider storing an original string as an extra property.

#### `<span-base-id>.<level>.<level>` mismatch

Some vendors may experiment with the `span-id` format. Use longer `<span-base-id>` or use characters other than base64 and `'.'` to record it. Protocol requires to apply **Reset** function to the strings like this. It is recommended that the original string is logged in some form - as a hashed value or as a string property with well-defined name.

#### Extra long header value

`Request-Id` header format defines the maximum size as 128 characters. If longer string received, `<trace-id>` format mismatch and `<span-base-id>.<level>.<level>` mismatch fallback rules must be applied in said order.


## The Correlation-Context Field

`Correlation-Context` is represented as comma-separated list of key value pairs, where each pair is represented in `key=value` format:

```
Correlation-Context: key1=value1, key2=value2
```

Keys and values MUST NOT contain `"="` (equals) or `","` (comma) characters.

Overall Correlation-Context length MUST NOT exceed 1024 bytes. Key and value length should stay well under the combined limit of 1024 bytes.

Note that uniqueness of the key within the Correlation-Context is not guaranteed. Context received from upstream service is read-only and implementation MUST NOT remove or aggregate duplicated keys.

# The Expected Environment for Using Request-Id

The fundamental use of a Request-Id is to link an activity that issues an HTTP
request with the code that runs (and produces logging messages) that services the
request.   It is useful to have an overview of how such systems typically work to frame
the design of the ID itself.  

The expectation is that the logging system has the concept like 
OpenTracing's Span,
which represents some work being done that has an ID as well as parent-child relationships
with other Spans.   Thus there are 'top level' Spans that do not have a parent, and every
other Span remembers the ID of its parent as well as its own ID.   Somewhere in the logging
system a table is maintained that allows a Span to be located given its ID.  Every 
logging message will also be marked with the Span that is currently active, and as a result
the logging system can group together all logging messages from any given Span (or any 
of its children).   

The requirements for the Request-ID for such an environment is pretty minimal.  It simply
needs to be unique to the Span.   The expectation is that such IDs are generated by 
choosing a random number from a large space of possible numbers.   Number sizes from 64 bits
(8 bytes) to 128 bits (16 bytes) have been commonly used.   

The expectation is that when some 'top level' activity is running, it will generate a 
new ID and log the necessary bookkeeping information for that Span.  There may be additional
Spans generated as part of the processing but eventually an HTTP request is made.  As part
of sending the HTTP request, a new Span (with a unique ID) is created (and logged), and that 
Span's ID is inserted into the Request-Id field of the HTTP request's header.  When the request 
is received a new Span (with at new ID) is created  with a parent ID as specified by the 
Request-Id field (and logged) to represent the server-side processing.   The whole sequence
can repeat in the case of nested requests.   

The result is that the logging system tags every message with the active Span, and every 
Span has an ID and its parent ID, which form a tree representing the causality of the 
execution of the multi-tiered application.  

## Simple Sampling with Request-Id

It is very likely that most systems want to implement some form of Sampling to avoid
overloading the logging system.   On simple way of doing this is to simply not emit the 
Request-Id header field in the request.   This is both simple and efficient (no overhead
in when a particular request is not being logged).   This works well when logging control
is given to the topmost activities in the system, but if this is not the case then you 
lose 'caller context' if you start tracing at some intermediate point.    Often systems 
wish to have this caller context available for rare logging (for example, when errors happen), 
which means that you need to send a Request-Id with all request.   Nevertheless this 
mechanism is useful and is easily supported.

## Improving on Flat IDs

While the system described above works, it has several disadvantages 

1. It requires a table that maps from ID -> Span that is large O(#Spans).  
2. If ANY entries in the table are missing (for example, because some components did not log properly), 
then some parent-child chain can be broken and cannot be recovered.   Effectively potentially large
chunks of the processing of a request might become 'orphaned' and not properly attributed to 
the top-level request that was responsible for it.  

These and other limitation can be avoided if IDs are given structure.   It is common to employ
a two-level structure where IDs have the form

```
TOP_LEVEL_ID - SPAN_ID
```

Basically all children of a given top-level Span have the same prefix.    With this ID structure
given any two Spans ID, they can be determined as belonging to the same top-level processing 
simply by comparing their IDs.   This has the following advantages

1. Is very fast
2. Does not need a global table 
3. Works in the presence of missing information on intermediate Spans.   

### Hierarchical IDs

The two level structure generalizes very naturally into a multi-level (hierarchical) ID.   Thus an
ID would have the form

```
TOP_LEVEL_ID - LEVEL_1_ID - LEVEL_2_ID - ... - LEVEL_N_ID
```

which creates ID for every Span by taking the parent's ID and adding a unique suffix.   With such 
a structure, it is possible to group things not only by the top-level Span but also every intermediate 
level as well.   

### Encoding Additional Information into an ID

The previous section shows how the parent-child relationship can be encoded into the ID itself.  
In general ANY information that is CONSTANT over the lifetime of the Span can be encoded into the ID 
as well without destroying its utility as an ID (which just requires uniqueness).   For example, the
start time, or start location could be part of the ID.   However doing this make the ID bigger and 
generally is not worthwhile unless the information is small and high value.   Candidates for such 
information high value, small size includes

1. Control information (like a verbosity bit (or bits))
2. Format and interpretation bits (for example, what version or semantic properties of the ID)

## Fixed (Bounded) Size IDs

Generally speaking fixed sized structures are easier to implement than variable sized structure.   
Thus there is non-trivial value if the IDs can be kept to a fixed maximum size.   Indeed many existing
logging systems mandate a particular size for the IDs (typically 64 bits or 128 bits).  

Note that systems that mandate fixed size IDs have a problem with hierarchical IDs (which have unbounded size).  
If a fixed-sized ID system receives an ID that is too large what can it do?   The solution is hashing.
Clearly a variable sized ID can be hashed to any particular fixed size without destroying its ability
to act as a unique ID.  However if some parts of the system hash (because they are fixed size), and other
do not, then in order to compare IDs it must be possible to look at both IDs, recognize that one is hashed
and then know how to perform the hash on the other before doing the comparison.   

# The Request-Id Field

The Request-ID is defined to be a variable length string of printable ASCII characters 
(from 0x20 through 0x7E inclusive).   This string uniquely identifies the request within 
the system as a whole, and is expected to be unique because some part of the ID was
chosen randomly from a large (64 bit or greater) number space.    Logging systems that
have only flat IDs will simply emit this ID as a string.   It is recommended that 
punctuation in the ASCII range 0x21-0x2A be reserved for use by this standard in the future.  

## Practical Impacts of the standard

It is useful at this point to give some examples of what practical impacts would be on various logging systems.

### A two tiered ID system with a fixed 16 bytes top ID and a fixed 8 bytes secondary ID

For this type of logging system, when writing the ID to the HTTP header, all that is necessary is to conform
to the syntax specified about.   This means encoding the two IDs using Base64 and outputting them separated
by a '-'.  

When reading the Request-Id field, instead of using a Base64 decoder that would fail on illegal inputs they
would look for the '-' in the ID, and use HASH_16 for the part before the '-' and HASH_8 for the id after
the dash.   These routines will be just as fast as a normal Base64 decoder. 

For the systems that also support the arbitrary properties collection the original string may be stored as `<trace-id>` or parent `<span-id>`.

### A multi-Tiered (variable sized) ID

A multi-level logging system should not have ID length limitations so it should accept the IDs without
modification.   The only thing such systems need to do is to follow the syntax for the multi-level ID
(for example, used '-' for the first level and '.' (termination) for all other levels).  


## Well Known Correlation-Context keys:

TODO COMPLETE: 

# Appendix

## Base64 encoding of binary blobs

This standard uses [base64](https://en.wikipedia.org/wiki/Base64) as a standard way of encoding binary blob as a sequence of printable ASCII characters. The characters used are confided to the alpha-numeric and two punctuation characters (`+` and `/`). Because there are 64 legal possible characters each one represents 6 bits of binary data. Base64 is a widely-used compromise between the length and human-readability of the resulting string.

## Hashing for Fixed Sized ID Systems

Systems that require the IDs to be a fixed size must hash IDs to fit into the required size (for example, 16 bytes or 8 bytes). This must be one with the hash algorithm defined here which we call HASH_N (where N is the number of bytes of the resulting binary blob). The key idea of the algorithm to be Base64 decoder modified to accept any input characters, and to circularly XOR the output bytes until the input is consumed (hash). This hash function has the following useful characteristics:

1. Its input can be anything string.
2. Its output is always N bytes.
3. When operating on Base64 encoded input that of size N, it produces the same result as simply decoding (no information loss).
4. It is as efficient as Base64 decoding.   

This mechanism has the following useful ramifications:

1. IDs from two like minded systems work without any loss of information.
2. Fallback algorithm for `<trace-id>` and `<span-base-id>` format mismatch can be coded in a single pass of a header value parsing.
3. ID has a useful comparison operator that works even in this mixed case. Basically if the IDs don't match perfectly, you also test if they match if HASH_N is applied to both (you may need to do this twice, once for the N of the first operand, and once for the N of the other operand).  
