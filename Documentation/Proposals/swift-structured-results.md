# 0001 - Swift Structured Results

This proposal is a follow up to a previous Pitch: [A Structured Approach to Build and Test Results Data](https://forums.swift.org/t/pitch-a-structured-approach-to-build-and-test-results-data/85799). Since the original Pitch, the team has been prototyping and researching the problem, which has informed this detailed proposal. Some of what was previously pitched has changed - those differences are summarized near the end of the document.

The vision for Structured Results is to provide a common set of APIs for recording the results of build, test, and other software development activities in a unified format with tooling for efficient and reliable processing of the data. This proposal is the first of several covering the first release of the package. It includes module layout, modelling approach, serialization, and on-disk layout.

## About This Proposal

This document describes the proposed design direction for Structured Results. It is not intended to be a definitive specification in the manner of an RFC for an open standard. It is published ahead of the pull requests that will implement the design, and its purpose is to establish the shape of the system. Where a detail is not fully specified here, it will be settled in the implementation and in later proposals, and Future Directions lists the known gaps. Where this document uses terms such as "must" or "invalid", it is describing the intended behavior of the implementation, not a conformance requirement on third parties. The versions in this proposal are `0.1.0` because the format may still change incompatibly. The Swift APIs described here should likewise be considered experimental at present. All versions in the package (the stream, layout, and built-in payload versions) will move to `1.0.0` together, and the APIs will become stable in coordination with that release.

## Overview

### Motivation

The problem space is covered in the original Pitch. The motivation for this effort, beyond that document, is that existing approaches suffer from one or more of several limitations:

- Lack of structured, open, and extensible data modelling
- Schema is highly specific to one problem domain (i.e. testing or building) or one implementation of one domain (e.g. Swift Testing's JSON stream)
- Lack of support for large, out of band, data attachments
- Unsuitable for providing a comprehensive "soup to nuts" representation of a developer job - not just a build log or tree of test results, but also the distributed interstitial activity that holds the job together

### Modules

Reflecting the producer/consumer nature of results data, the first phase of this project includes two primary modules - a "Recorder" and a "Reader" - as well as a command line tool:

- `StructuredResultsRecorder` - module imported by producers of results data with APIs for recording results and writing the data to a directory.
- `StructuredResultsReader` - module imported by consumers of results data with APIs for reading and querying results data.
- `ssrtool` - a command line interface to the reader module's capabilities.

This proposal does not describe the Reader module and the CLI in detail. A separate document will propose their design, including the aggregation and analysis capabilities discussed in the Pitch, such as summarizing outcomes, failures, and durations across a session.

### Modelling

In Structured Results, data is primarily represented as streams of events, organized in a tree hierarchy, from one or more producers. Each event carries a payload which can be one of a set of core types defined in the package, a user defined type, or an extension of either a core or user type. In addition to the payload, all events have a universally unique identifier, a reference to a parent event, and a payload type that identifies the type of their payload. Optionally, events can have timestamps as well. In addition to events and their payloads, results can include attachments, which are files or other data that are referenced from the event stream but stored separately. Examples include media (image, audio, video, etc.) and other forms of binary data.

Multiple processes can record event streams and attachments as part of an overall “session” of structured results. A developer action such as building and running tests for a project would produce a results session containing event streams for compilation, testing, and related activities such as installing executables and launching processes. The individual streams can be organized in a shared directory to provide a single artifact with all data for a session, or the content can be decomposed and stored according to the requirements of the system adopting it.

### Schemas

Structured Results provides both recording and reading implementations that can be adopted into other projects, but it also allows for interoperability with software that cannot call into Swift. Structured Results will do that by providing JSON Schema specifications for all types that the recorder writes, including the event stream header, events, the predefined payloads, and the artifact manifest. This will enable other systems to create their own recording and reading implementations. Tools that convert previously collected data, which may lack timestamps, should write event streams directly against these schemas rather than through the Recorder, since passing no timestamp to the Recorder results in the current time being recorded. In addition, the `ssrtool` utility can format its output in JSON and will have specifications for that schema as well. The schemas will be published alongside the implementation, as a companion to the description in this proposal.

### Serialization

Structured Results writes the event stream in a [JSON Lines](https://jsonlines.org) format. Payloads must support encoding. Events are written to a local file as quickly as possible to minimize data loss in the case of a producer crash. No affordance is provided within the recorder module for any other destination (network, IPC, etc.) to which the event stream can be written. This is an intentional design choice to maximize recorder reliability. Systems which need to capture and transport the data to destinations other than a local file need to implement a separate collector process for that purpose. At present, JSON Lines is the only planned serialization approach for the event stream, but the design does not preclude future alternatives.

### Considerations

We want Structured Results to be suitable for adoption by results data producers at all levels of the stack. Data producers should be able to have a direct dependency on the Recorder module, call its APIs, extend its payloads, etc. This puts restrictions on the API of the Recorder module. For example, Swift Testing records its results through the Recorder, and Foundation's own tests use Swift Testing. If the Recorder's API exposed Foundation types, Swift Testing would depend on Foundation through that API, creating a dependency cycle when testing Foundation. As a result, the APIs of the Recorder module use only standard library types, for example passing data such as UUIDs using their string representations.

This is a distinction about the API, and the Recorder's use of Foundation internally is also limited. The Recorder imports Foundation only behind `#if canImport(Foundation)`, so that building the Recorder doesn't require Foundation to be present. On Linux and Windows it uses FoundationEssentials rather than the full Foundation. Where Foundation is available, the Recorder uses it for work such as JSON encoding and file operations. Where it isn't, for example when Foundation itself is being built and tested, the Recorder uses a minimal custom `Encoder` that writes JSON, and direct system calls, instead, so that it still records results. Both paths produce the same stream and artifact format described in this proposal.

The Reader module and the CLI are not expected to have the same constraints as the Recorder.

### Requirements

This proposal targets Swift 6 and the following platforms: macOS 14.0, iOS 17.0, tvOS 17.0, watchOS 10.0, visionOS 1.0, and Mac Catalyst 17.0, each and later, as well as Linux and Windows.

---

## Detailed Approach

The remainder of the document describes the primary concepts of Structured Results, how the Recorder module models them, and how they are stored on disk. They are:

- Events - the primary unit of results information, wrapping a payload of data with the structure of the stream.
- Payloads - the actual results data (vs the structure in the event wrapper). Payloads are extensible.
- Attachments - data referred to from the event stream but stored separately.
- Sessions - the sum of all results data for a job, not implemented as a formal type, but uniquely identified.
- Channels - discrete streams of results. A session may contain many channels from different processes and hosts.
- Artifact Layout - how results data is organized on disk.

Producers record results through the Recorder's `Channel` class: they create a channel, start it, send events and save attachments through it, and finalize it. The Swift declarations of the Recorder API appear in the sections describing the concepts they model. Examples in the earlier sections use a `channel` before the Channels section describes it in detail.

Results data involves many modelling details for various domains such as building and testing. This proposal is focusing on the core design of Structured Results. A subsequent proposal will describe a series of payloads and utility structures for data modelling. Any examples of such types in this proposal are present to facilitate explanations of the core system and aren't part of this proposal.

---

### Event

An Event is the fundamental unit of the results stream. Everything a producer reports is recorded as an event – the start of a test, a diagnostic, an attachment, etc. All events have a set of core properties paired with an extensible payload.

#### Event Identifiers

Events must be universally uniquely identified. This ensures that results data from different sessions can be safely and cheaply merged and guarantees that the event tree is well formed and does not contain collisions. Event ids are UUIDs represented as strings. All UUIDs in Structured Results (event identifiers, session identifiers, channel identifiers, and the file names derived from them) are written in lowercase canonical form, `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`. Producers must write lowercase, so consumers can compare identifiers as exact strings.

#### JSON Structure

```json
{
  "id": "b026ad51-ecc0-419d-a3d1-8f81f78e986f",
  "parentID": "3a594587-86c6-426f-86cc-24c3a95d3e0c",
  "timestamp": 1757548800.123,
  "monotonicTime": 41234.5,
  "payloadType": [{ "name": "com.example.MyCustomEventPayload" }],
  "payload": { "customFieldFoo": ["bar", "baz"] }
}
```

- `id` - Identifier for the event, the string representation of a UUID.
- `parentID` - The identifier of this event's parent: either another event's id or, for a top-level `Channel` event, the session identifier.
- `timestamp` - Wall-clock time in seconds, Unix epoch, double precision floating point. Optional.
- `monotonicTime` - Reading from a monotonic clock, in seconds, double. Arbitrary epoch. Optional.
- `payloadType` - Payload type, an array that describes the type ancestry, from the root (base) payload first to the most specific last.
- `payload` - The typed body. Its shape is determined entirely by `payloadType`.

An event carries these fields and nothing else. Anything a producer wants to add belongs in the payload. Consumers ignore any unrecognized fields in an event, and likewise in the stream header, so later versions of the stream can add fields. (The first line of an event stream is a header, not an event; see Stream Encoding.)

In the Recorder, an event is an `Event`:

```swift
public struct Event: Encodable, Sendable {
    public typealias ID = String

    public let id: ID
    public let parentID: ID
    public let timestamp: Double?
    public let monotonicTime: Double?
    public var payloadType: PayloadType { get }
    public let payload: any Payload

    public init(
        id: ID? = nil,
        parentID: ID,
        timestamp: Double? = nil,
        monotonicTime: Double? = nil,
        payload: any Payload
    )
}
```

If `id` is omitted, a new lowercase UUID is generated. `payloadType` is derived from the payload's type (see Payload Type). Omitted clock readings are filled in as described under Two Timestamps.

#### Two Timestamps

Events have two timestamps, both optional because there are some very limited systems in which there may be no clock available or because the data is representing content which was previously collected without timestamps.

- `timestamp` is what a person reads and what correlates the stream against data from other systems. On most platforms it's also adjustable - NTP steps and manual clock changes can make it run backward or jump ahead mid-run. This means that the timestamp is less reliable for ordering or offset/duration computations within a channel. Timestamps from different hosts are, of course, even more likely to have drift.
- `monotonicTime` never runs backward. Its epoch is arbitrary and only comparable within a single producer's process, so it isn't suitable for correlation but is reliable for ordering and measuring duration. To find out how long something took, subtract two monotonic time readings. To find out when something happened, with respect to other sources of data, read `timestamp`.

When an `Event` is created, any clock reading the producer doesn't supply is captured at that moment. For `timestamp`, the recorder reads the system's wall clock and stores the value in the event. For `monotonicTime` there are two candidate clocks, and the choice between them belongs to the channel, so the event captures a reading from each and holds them privately: one from a clock that doesn't advance while the system sleeps (Swift's `SuspendingClock`) and one from a clock that does (`ContinuousClock`). When the event is handed to the channel with `send(event:)`, the channel selects the reading from the clock it was configured with, so that every event in the channel uses the same clock. The selected value was captured when the event was created, not when it was sent.

Events the channel records itself, such as the completions of spans it closes when it is finalized, are created and handled the same way. In an environment where a clock isn't available, the reading it would provide is omitted from the event. Channels use `SuspendingClock` by default, so that time spent asleep doesn't count toward durations such as test runs. A producer that supplies its own `monotonicTime` must read it from the channel's configured clock, because readings from different clocks can't be compared.

Until an event is sent, its `monotonicTime` property is `nil` unless it was supplied explicitly, and an `Event` encodes only the values it holds. The two captured readings are not exposed. When the channel records the event, it writes a copy whose `monotonicTime` is the reading it selected from the two captured values; the caller's `Event` is unchanged.

#### Event Tree

Every event has a parent. Parents define the stream's structure, which a consumer can reconstruct. For example:

```
session
└─ channel
   ├─ custom task
   │  └─ diagnostic collected
   └─ channel
      └─ run tests
         ├─ run test A
         └─ run test B
            ├─ test failed assertion
            └─ attachment triage data
```

An `Event`'s parent must be an event in the same stream unless it is a `Channel` event. An event must appear in the stream after its parent, so a consumer reading a stream in order has always seen an event's parent before the event itself. This also means a `Completion` follows the event that opened its span, and an `AttachmentFinalization` follows its `Attachment`. A `Channel` event's parent is in another stream or is the session identifier, so this ordering doesn't apply to it. Beyond that, any event can have child events that refer back to it as their parent without a need for any additional designation.

#### Spans and Instant Events

Some events describe the beginning of an action which will formally complete later with an explicit success or failure outcome. For example, a test case starts running or a compiler begins building a unit of code. Such events are referred to as spans. Other events are considered instantaneous, for example a diagnostic was logged or an attachment was saved, even if they have descendants that occur later in time.

A span opens with an event whose payload is `Start` or an extension of `Start`, and that event's `id` identifies the span. It is formally closed when a `Completion` event is recorded whose parent is the opening event. The completion has an outcome which indicates if the span `succeeded`, `failed`, or was `interrupted`. If the span's completion was recorded, its duration is the time elapsed between the start and the completion, measured with `monotonicTime` if both events have it (preferred), and otherwise with `timestamp` if both have it. In cases where the completion was never recorded, the span is interpreted by consumers as interrupted and its duration is undefined. This applies to any span without a `Completion`, including one in a channel that otherwise finished. Examples for interruption include cancellations and timeouts. How consumers recognize `Start` and `Completion` events, including extensions of them, is described under Predefined Payloads.

A span has at most one `Completion`. A second `Completion` whose parent is the same span is invalid.

The Recorder declares the span payloads and the outcome as:

```swift
public struct Start: Payload, Sendable {
    public static let payloadTypeName = "Start"
    public static let version: String? = "0.1.0"

    public let message: String?

    public init(message: String? = nil)
}

public struct Completion: Payload, Sendable {
    public static let payloadTypeName = "Completion"
    public static let version: String? = "0.1.0"

    public let outcome: Outcome
    public let message: String?

    public init(outcome: Outcome, message: String? = nil)
}

public enum Outcome: String, Codable, Sendable {
    case succeeded
    case failed
    case interrupted
}
```

A producer opens a span by sending a `Start` event and closes it by sending a `Completion` whose parent is that event:

```swift
let build = Event(parentID: channel.channelID, payload: Start(message: "Building and testing project XYZ"))
try channel.send(event: build)
// ... perform the work ...
try channel.send(event: Event(parentID: build.id, payload: Completion(outcome: .succeeded)))
```

This records events like the following (`monotonicTime` omitted for brevity):

```json
{"id":"a1f3c0de-5b2e-4c7a-9d10-3e8f6b2a7c41","parentID":"c0ffee00-1b0a-4998-8877-665544332211","timestamp":1757548800.0,"payloadType":[{"name":"Start","version":"0.1.0"}],"payload":{"message":"Building and testing project XYZ"}}
{"id":"b2e4d1f0-6c3a-4d8b-8e21-4f9a7c3b8d52","parentID":"a1f3c0de-5b2e-4c7a-9d10-3e8f6b2a7c41","timestamp":1757548801.4,"payloadType":[{"name":"Completion","version":"0.1.0"}],"payload":{"outcome":"succeeded"}}
```

#### Stream Encoding

The stream is JSON Lines: one record serialized to a single line of UTF-8, terminated by `\n`. There's no enclosing array and no commas between records. Each line is independently parseable, so a consumer can start processing the stream while the producer is still appending to it.

Optional properties without a value are omitted. Consumers treat a property whose value is `null` as absent. This applies throughout the format, including the stream header, events, payloads, and the manifest.

The first line of every stream is a header. It is a JSON object but is **not** an Event: it has none of the Event fields and is never part of the event tree. Every subsequent line is an Event.

```json
{"ssrStreamVersion":"0.1.0","sessionID":"5e551011-4455-4677-8899-aabbccddeeff","channelID":"c0ffee00-1b0a-4998-8877-665544332211"}
```

- `ssrStreamVersion` - Version of the event stream format, a version string conforming to [Semantic Versioning](https://semver.org) (SemVer). Required.
- `sessionID` - The identifier of the session the stream belongs to, the string representation of a UUID. For a nested channel this is the session of the enclosing channel, not the `parentID` of the `Channel` event. For a top-level channel it equals the `Channel` event's `parentID`. Required.
- `channelID` - The identifier of the channel that wrote the stream, the string representation of a UUID. Once the channel is started, it is the `id` of the stream's `Channel` event. Required.

The Recorder writes the header when a `Channel` is created (see Lifecycle).

A consumer reads the header first and uses `ssrStreamVersion` to decide how to decode the remaining lines. If a stream has any terminated line, its first terminated line must be a valid header, otherwise the stream is invalid.

The header is written atomically: the complete line, including its terminating `\n`, is written in a single operation. A stream written by the Recorder therefore contains either a complete header or nothing. A zero-length stream, such as one left by a crash during Init, is valid and contains no data.

A final line that is not terminated by `\n` is the result of truncation. Consumers discard it and process every preceding line normally. This applies to the header as well: consumers must tolerate a truncated header, for example from another writer or an interrupted copy, and treat it as an empty stream.

The header is the authoritative source of a stream's identity. A stream may be stored in something other than a file named for the channel, or outside an artifact entirely, so consumers must rely on the header, not on a file name or artifact membership, to determine the stream's channel and session.

#### Versioning

The event stream is versioned and follows SemVer. The version at the time of this proposal is `0.1.0`, and it is recorded in the `ssrStreamVersion` field of each stream's header. The event stream version is distinct from the versions of various payloads and from the artifact layout version. It governs the header, the fields of the event itself, and the encoding of the event stream. It is possible for a session of results data with multiple producers to contain more than one stream version, since each stream declares its own. Structured Results uses one compatibility rule for every version in the format - stream, layout, and payload versions: two versions are compatible if they are identical strings, or if they have the same major version and, for major version 0, the same minor version, and neither is a pre-release version. A version that isn't SemVer, or an omitted version, is only compatible with an identical one. A consumer can read a stream whose version is compatible with one it supports.

---

### Payload

All variation in the stream is provided by the payload – the typed body an event carries. The format is open by design so that a vendor can define new payloads as well as extend existing ones (see Payload Type).

Examples use reverse-DNS names under `com.example.` and `org.example.` for vendor payloads. In prose and tables, those names are shortened by omitting the prefix, so `com.example.Issue` is written `Issue`.

`Payload` is a protocol that any `Codable`, `Sendable` type can conform to. Its additional requirements identify the type:

```swift
public protocol Payload: Codable, Sendable {
    static var payloadTypeName: String { get }
    static var version: String? { get }
}
```

`payloadTypeName` is the `name` of the type's component in `payloadType`, and `version`, when not `nil`, is its `version`.

Every payload declares `version`, using `nil` for a payload without a version:

```swift
static let version: String? = "1.0.0"   // a versioned payload
static let version: String? = nil       // a payload without a version
```

A payload must encode as a JSON object, because extended payloads nest their base payload inside an object and consumers preserve undecoded data by property name (see Type Resolution).

Payloads can extend other payloads. A payload that extends another conforms to `ExtendedPayload`, which adds a `base` property holding an instance of the payload being extended:

```swift
/// Allows declaring custom payloads that extend known payloads.
///
/// An extended payload is serialized as `{"base": <base payload>, <additional data>}`.
public protocol ExtendedPayload<BasePayload>: Payload where BasePayload: Payload {
    associatedtype BasePayload

    /// The payload being extended.
    var base: BasePayload { get }
}
```

A payload that doesn't extend anything conforms only to `Payload` and only serializes its own properties. An extended payload nests its base payload under the key `base` and uses that key for nothing else; its own properties must have other names.

#### Payload Type

`payloadType` is an array of components that describes a sequence of payload extensions - A is extended by B which is extended by C.

```json
{
  "payloadType": [
    { "name": "com.example.Test" },
    { "name": "com.example.AnnotatedTest", "version": "2.0.0" }
  ]
}
```

Each component has a `name` (required) and an optional `version`. The type array reads as a chain: the above payload is an `AnnotatedTest`, which extends a `Test` type. A payload can't extend itself, even a different version of itself, so the component names within a `payloadType` are unique. This also keeps undecoded data, which is grouped by component, unambiguous (see Type Resolution). Because `payloadType` lists a payload's full chain of types, a consumer can decode the layers it knows and preserve the rest: a consumer that doesn't know `AnnotatedTest` but does know `Test` reads the payload as a `Test` and keeps the remaining data as raw JSON. This is what makes the format open: vendors can extend payloads without breaking consumers that do not have access to their extensions.

Payload names that don't contain a period (`.`) are reserved for Structured Results, including the predefined payloads (see Predefined Payloads) and payloads defined by future proposals. The name of any other payload must contain a period. Vendors that define payloads are recommended to use a reverse-DNS prefix they control, such as `com.example.`.

In the Recorder, a payload type is represented by `PayloadType`, and every type conforming to `Payload` provides its own through `payloadType`:

```swift
public struct PayloadType: Codable, Sendable, Hashable {
    public struct Component: Codable, Sendable, Hashable {
        public let name: String
        public let version: String?
    }

    public let components: [Component]
}

extension Payload {
    public static var payloadType: PayloadType { get }
}
```

A `PayloadType` encodes as the bare array of components shown above. For a payload that conforms only to `Payload`, `payloadType` has a single component built from its `payloadTypeName` and `version`. The payload type of an extended payload is formed by appending its component to the payload type of its `BasePayload`. For example, the predefined `Channel` payload is an `ExtendedPayload` whose `BasePayload` is `Start`, so its payload type is `[Start, Channel]`.

#### Payload Extension

Given a base payload `SomeDiagnostic` whose `payloadTypeName` is `com.example.SomeDiagnostic`, JSON might be:

```json
{
  "payloadType": [
    { "name": "com.example.SomeDiagnostic" }
  ],
  "payload": {
    "severity": "failure",
    "message": "Assertion failed"
  }
}
```

Another vendor might then extend `SomeDiagnostic` with its own payload, `org.example.TriagedDiagnostic`, to capture additional fields for its use case:

```json
{
  "payloadType": [
    { "name": "com.example.SomeDiagnostic" },
    { "name": "org.example.TriagedDiagnostic", "version": "1.5.0" }
  ],
  "payload": {
    "base": {
      "severity": "failure",
      "message": "Assertion failed"
    },
    "triageOwner": "space-team",
    "knownIssueID": "12345"
  }
}
```

Each extension composes a new layer of nested data. A consumer that has not registered a type for `TriagedDiagnostic` but has registered `SomeDiagnostic` will decode the value of `base` as a `SomeDiagnostic`. The remaining properties, `triageOwner` and `knownIssueID`, will be available as raw JSON, grouped under the `TriagedDiagnostic` layer (see Type Resolution).

Extensions can be stacked to any depth. Given a base payload `Issue`, `TaggedIssue` extends it with tags, and `ContextualIssue` extends `TaggedIssue` with environment data:

```swift
struct Issue: Payload {
    static let payloadTypeName = "com.example.Issue"
    static let version: String? = nil

    var filePath: String
    var lineNumber: Int
    var message: String
}

struct TaggedIssue: ExtendedPayload<Issue> {
    static let payloadTypeName = "com.example.TaggedIssue"
    static let version: String? = nil

    var base: Issue
    var tags: [String]
}

struct ContextualIssue: ExtendedPayload<TaggedIssue> {
    static let payloadTypeName = "com.example.ContextualIssue"
    static let version: String? = nil

    var base: TaggedIssue
    var environment: [String: String]
}
```

An event carrying a `ContextualIssue` is written with one level of `base` nesting per layer:

```json
{
  "payloadType": [
    { "name": "com.example.Issue" },
    { "name": "com.example.TaggedIssue" },
    { "name": "com.example.ContextualIssue" }
  ],
  "payload": {
    "base": {
      "base": {
        "filePath": "Sources/App/main.swift",
        "lineNumber": 42,
        "message": "Unused variable 'x'"
      },
      "tags": ["lint", "warning"]
    },
    "environment": { "CONFIGURATION": "debug" }
  }
}
```

Because `base` is a requirement of `ExtendedPayload`, a type that already has a property named `base` can't conform directly. To use such a type to extend a base payload, create a wrapper that conforms to `ExtendedPayload` and holds the existing type as a property next to `base`:

```swift
struct LegacyTriagedDiagnostic: ExtendedPayload<SomeDiagnostic> {
    static let payloadTypeName = "com.example.LegacyTriagedDiagnostic"
    static let version: String? = "1.0.0"

    let base: SomeDiagnostic
    let triage: ExistingTriageData   // preexisting type with its own `base` property
}
```

This serializes as `{"base": {…}, "triage": {…}}`, so the existing type's `base` stays nested inside `triage` and doesn't collide.

#### Type Resolution

A consumer holds a registry of the payloads it can decode, with at most one registered type for each name and version. For each event, it resolves the payload as follows:

1. Look up the event's full `payloadType` in the registry. A registered type matches a payload type when the registered type's own `payloadType` has the same number of components, and each pair of corresponding components has the same name and compatible versions, as defined under Versioning.
2. If more than one registered type matches, prefer, by the version of the last component, in order: the same version; otherwise the closest newer version, that is, the lowest compatible version higher than the event's; and otherwise the closest older version, the highest compatible version lower than the event's. A newer version is preferred because it represents the data more fully, according to the intent of the payload's author, instead of leaving some of it as unrecognized properties. This relies on properties added in a minor version being optional or having default values (see Payload Type Versioning).
3. If no registered type matches, drop the last component and look up the shorter payload type, repeating until a registered type matches or no components remain.
4. When a registered type matches, decode the corresponding layer of the payload as that type. Decoding of that layer, including its base layers, is delegated entirely to the matched type, so its base types don't need to be registered separately. The layers belonging to dropped components are left undecoded and preserved as raw JSON, as described below.
5. If no components remain, no layer is decoded, and every layer is preserved as raw JSON, as described below.

For example, consider an event whose `payloadType` is:

```json
[{"name": "com.example.Report", "version": "2.0.0"}, {"name": "com.example.TimedReport", "version": "1.1.0"}]
```

and a consumer whose registry contains three types, each shown by its own payload type:

```
[com.example.Report 1.3.0, com.example.TimedReport 1.0.0]
[com.example.Report 2.1.0]
[com.example.Report 2.4.0]
```

Resolution proceeds as follows:

| Step | Payload type | Registered candidates | Result |
|---|---|---|---|
| 1 | `[Report 2.0.0, TimedReport 1.1.0]` | `[Report 1.3.0, TimedReport 1.0.0]` | No match: `TimedReport 1.0.0` is compatible with `TimedReport 1.1.0`, but `Report 1.3.0` isn't compatible with `Report 2.0.0` |
| 2 | `[Report 2.0.0]` | `[Report 2.1.0]`, `[Report 2.4.0]` | Match: `[Report 2.1.0]`, the closest newer version |

The consumer decodes `payload.base` as `Report 2.1.0`, and the properties added by `TimedReport` are preserved as raw JSON under the `TimedReport 1.1.0` component.

Unwrapping is in complete layers: a type that matches after `k` components have been dropped decodes the value reached by unwrapping `base` `k` times.

Undecoded data is preserved as raw JSON, grouped by layer and keyed by each layer's payload type component, that is, its `name` and `version`, for each layer that has undecoded data. The value holds that layer's own properties that were not decoded, excluding `base`. For a dropped layer that is all of its properties; for a layer the decoder does handle, it is any properties the decoder doesn't know, such as ones added by a newer compatible version. Layers with nothing undecoded are omitted. In the Reader, this data is exposed on each event as `unrecognizedProperties`, a dictionary from `PayloadType.Component` to JSON; the representation of JSON values is left to the Reader proposal.

What a consumer decodes from the `ContextualIssue` event shown under Payload Extension depends on which of its types it has registered:

| Consumer has a decoder for | Decodes | Undecoded layers (raw JSON) |
|---|---|---|
| `ContextualIssue` | `payload` | none |
| `TaggedIssue` | `payload.base` | `ContextualIssue` |
| `Issue` | `payload.base.base` | `TaggedIssue`, `ContextualIssue` |
| none of them | nothing | `Issue`, `TaggedIssue`, `ContextualIssue` |

A consumer that knows only `Issue` receives the undecoded layers keyed by the component that added them. These example types have no versions, so each key is shown by its name:

```json
{
  "com.example.TaggedIssue": {
    "tags": ["lint", "warning"]
  },
  "com.example.ContextualIssue": {
    "environment": { "CONFIGURATION": "debug" }
  }
}
```

#### Payload Type Versioning

Each payload type component has an optional `version`, and type resolution compares both name and version, using the compatibility rule described under Versioning. It is strongly recommended that the version string conforms to SemVer, which lets a consumer decode events of a different but compatible version of a type it knows: a consumer with a decoder for type `X` at version `1.1.0` can decode `X` at version `1.0.0` or `1.2.0`. To make this work, a field added to a payload in a minor version must be optional or have a default value. When an extended payload moves to a different version of its base payload, its own version should change too: a minor version change is enough if the extension's own properties are otherwise unaffected.

#### Predefined Payloads

This proposal includes these predefined payloads, each at version `0.1.0`:

- `Start` - Opens a span. Carries an optional message. The event's id identifies the span. See Spans and Instant Events.
- `Completion` - Closes its parent span (as specified by its parent id). Has an outcome, and an optional message. See Spans and Instant Events.
- `Channel` - Root of a producer's stream, and a span. An `ExtendedPayload` of `Start`. Its Swift type is `ChannelPayload`; see Channels.
- `Attachment` - Declares an attachment with a `fileName`. The `fileName` must not contain path separators (`/` or `\`); an `Attachment` whose `fileName` does is invalid. See Attachments.
- `AttachmentFinalization` - Finalizes a previously declared attachment (see Attachments). Its parent is the `Attachment` event being finalized. Exactly one of the following properties is present:
  - `size` - the attachment was saved, and this is its size in bytes. A size of 0 is valid and denotes an empty attachment.
  - `discardReason` - the attachment was discarded and no data was saved. The value is a human-readable string explaining why.

  An `Attachment` has at most one `AttachmentFinalization`. A finalization that has both properties or neither is invalid.

Consumers recognize the predefined structural payloads by the names of the leading components of `payloadType`, regardless of version and of any further components: an event opens a span if its first component is `Start`; it is a channel if its first two components are `Start` and `Channel`; it is a completion if its first component is `Completion`; and likewise for `Attachment` and `AttachmentFinalization`. Extensions of these payloads therefore retain their structural meaning, and recognizing them doesn't depend on whether a consumer can decode the payload.

Additional payloads that more specifically model various categories of results data such as build and test activities will be part of future proposals.

---

### Attachments

Attachments are used for data that is not directly recorded in the stream. Reasons for using an attachment to record data instead of an event payload include:

- The data is not easily modelled as a payload. This is true for most binary formats such as image and other media files. They can be encoded into JSON using base 64, but for most use cases this will not be advantageous.
- The data is relatively large. Without specifying a precise limit, large event payloads will impact processing, transfer, and storage. Using an attachment for larger content keeps the stream lightweight.
- The data is repeated in the results. Depending on the size and how many times it repeats, it could be more efficient to record the data once as an attachment and have multiple references to that attachment throughout the stream. An affordance for referencing previously created attachments will be part of a future proposal.

An attachment is reported in two events:

1. The first event, `Attachment`, is recorded explicitly by the producer. It places the attachment in the event tree with a `fileName` property. If that filename has a file extension it may be used by consumers to infer the type of data in the attachment. This provides enough information for a consumer to render a placeholder if needed and also makes it easy for the producer to control where the attachment event is located in the event tree even if the data for the attachment is not ready. For example, a test that manipulates the UI might want to record the screen during the test but have the attachment for that recording appear at the very beginning of the test, even though the recording won't be finished until the test has completed.
2. Subsequently, an `AttachmentFinalization` event is recorded to do one of two things:
    1. indicate the attachment was saved and the `size` in bytes of the attachment. When the producer saves the attachment's data through the channel, the channel records this finalization itself. It is not recorded until the attachment's data has been fully written, so a finalization with a `size` guarantees the data is complete. If saving fails, the channel removes any partially written attachment file, records a finalization with a `discardReason` describing the failure, and then `save` throws.
    2. indicate that the attachment was discarded, which might happen if the test had a policy of not preserving large diagnostic assets when tests have passed. A producer that discards an attachment instead of saving it records this finalization explicitly, with its `discardReason`.

An `Attachment` without a subsequent `AttachmentFinalization` is treated by consumers as interrupted, in the same way as a span without a `Completion`. Consumers must not assume its data exists or is complete.

The Recorder declares the attachment payloads as:

```swift
public struct Attachment: Payload, Sendable {
    public static let payloadTypeName = "Attachment"
    public static let version: String? = "0.1.0"

    public let fileName: String

    public init(fileName: String)
}

public struct AttachmentFinalization: Payload, Sendable {
    public static let payloadTypeName = "AttachmentFinalization"
    public static let version: String? = "0.1.0"

    public let size: Int?
    public let discardReason: String?

    public init(discardReason: String)
}
```

Only discard finalizations can be created by producers. Finalizations with a `size` are recorded by the channel's `save` methods, which write the attachment's data into the artifact (see Attachment Files) and record the `AttachmentFinalization` before returning:

```swift
extension Channel {
    public func save(attachmentWithID attachmentID: Event.ID, bytes: [UInt8]) async throws
    public func save(attachmentWithID attachmentID: Event.ID, buffer: UnsafeRawBufferPointer) async throws
    public func save(attachmentWithID attachmentID: Event.ID, filePath: String) async throws
}
```

The data being saved must remain available until `save` returns: the memory referenced by `buffer` must stay valid, and the file at `filePath` must stay in place and unchanged.

For example, a test that records the screen declares the attachment when the test starts and then either saves or discards the recording when the test ends:

```swift
let recording = Event(parentID: test.id, payload: Attachment(fileName: "screen-recording.mov"))
try channel.send(event: recording)
// ... the test runs while the screen is recorded ...
if keepRecording {
    try await channel.save(attachmentWithID: recording.id, filePath: recordingPath)
} else {
    let discard = AttachmentFinalization(discardReason: "Test passed; recording not kept")
    try channel.send(event: Event(parentID: recording.id, payload: discard))
}
```

---

### Sessions and Channels

Results data is organized into sessions, each corresponding to a significant developer activity, such as a CI job or a local build and test plan of a project. Sessions contain at least one producer and may contain many producers in separate processes or distributed across multiple hosts. Each producer records results data using a Channel, which provides APIs for both events and attachments. While there is no formal type for sessions, a session must have a universally unique identifier (UUID). When work starts, the session identifier should be created and propagated to every producer that will contribute data. There is no session event.

```
session                       one result set, for example a CI job or a `swift test` invocation
├─ channel (orchestration)    one producer process
│  ├─ channel (build system)
│  │  └─ ...
│  └─ channel (test runner)   a nested producer, for example a test runner on a device
│     └─ test (plan)
│        └─ ...
└─ channel (script)           an independent producer in the same session
   └─ ...
```

#### Channels

Processes that record results must instantiate a `Channel`, whose corresponding payload, named `Channel` with the Swift type `ChannelPayload`, extends the `Start` payload. A channel is restricted to a single producer process: only that process writes the channel's stream. A process may open more than one channel, for example to report independent streams of work. A channel's stream begins with the stream header, followed by a `Channel` event as the first event; the `Channel` event's `id` is the channel identifier and equals the header's `channelID`.

```swift
public final class Channel: Sendable {
    public let channelID: Event.ID

    public init(
        sessionID: Event.ID,
        channelID: Event.ID? = nil,
        parentID: Event.ID? = nil,
        artifactPath: String,
        monotonicClock: MonotonicClock = .suspending
    ) throws

    public func start(startTimestamp: Double? = nil, monotonicTime: Double? = nil) throws
    public func send(event: Event) throws
    public func finalize(
        endTimestamp: Double? = nil,
        monotonicTime: Double? = nil,
        outcome: Outcome = .succeeded
    ) async throws
}

public enum MonotonicClock: Sendable {
    case suspending
    case continuous
}
```

- `sessionID` - The session the channel belongs to.
- `channelID` - The channel identifier, which becomes the `id` of the channel's `Channel` event. When it's `nil`, the channel generates a new lowercase UUID. Either way, the identifier is available through the `channelID` property.
- `parentID` - The parent of the `Channel` event. `nil` makes the channel top-level, with `sessionID` as its parent (see Hierarchy).
- `artifactPath` - The artifact directory the channel writes to (see Artifact Layout).
- `monotonicClock` - The clock whose reading is recorded for events that don't supply their own `monotonicTime`: `SuspendingClock` (`.suspending`, the default) or `ContinuousClock` (`.continuous`). See Two Timestamps.

`start(startTimestamp:monotonicTime:)` records the channel's `Channel` event; its parameters are that event's `timestamp` and `monotonicTime`, and readings that are omitted are taken as for any other event. `finalize(endTimestamp:monotonicTime:outcome:)` records the channel's own `Completion` with `outcome`, using the given clock readings or, when they're omitted, current ones. The events `finalize` records for unfinalized attachments and open spans use current readings.

The `save` methods for attachment data are described under Attachments, and how the methods map to the channel's stages under Lifecycle.

Here is an example of the `Channel` event (`monotonicTime` omitted for brevity) and a description of the fields in its payload:

```json
{
  "id": "c0ffee00-1b0a-4998-8877-665544332211",
  "parentID": "5e551011-4455-4677-8899-aabbccddeeff",
  "timestamp": 1757548800.0,
  "payloadType": [{"name": "Start", "version": "0.1.0"}, {"name": "Channel", "version": "0.1.0"}],
  "payload": {
    "base": {},
    "processID": 24071
  }
}
```

- `processID` - The producer's process identifier, for example a Unix or Windows process ID. Optional.

The channel creates the payload when it starts and sets `processID` to the producer's process ID, from `getpid()` on Unix or `GetCurrentProcessId()` on Windows. `processID` is an `Int` so that it can hold either: a Windows process ID is an unsigned 32-bit `DWORD`, which doesn't always fit in an `Int32`.

```swift
public struct ChannelPayload: ExtendedPayload<Start>, Sendable {
    public static let payloadTypeName = "Channel"
    public static let version: String? = "0.1.0"

    public let base: Start
    public let processID: Int?
}
```

Every other event recorded by that channel is a descendant of its `Channel` event. A consumer can therefore attribute any event to the channel, and so the producer, that reported it by walking up the tree to the nearest `Channel` event.

#### Hierarchy

The parent of a Channel event determines where the channel sits in the session's extended event tree:

- Top-level channel. The parent is the session identifier. Use a top-level channel for a producer that's independent of the others in the session - for example, an archiving script that runs after a build.
- Nested channel. The parent is an event in another channel. Use a nested channel when one producer launches or orchestrates another, so that the nested producer's results appear under the work that invoked them.

The parent of a nested channel can be the enclosing Channel event itself, or any event within the enclosing channel. For example, a test harness that launches a test runner on a device might parent the runner's channel to its own channel span:

```
session
└─ channel             xcodebuild on the host Mac
   └─ channel (test)   test runner on an iPhone
      └─ test (plan)
         └─ test (suite)
            └─ ...
```

Channels can nest to any depth. The enclosing producer passes the session identifier and the parent event's id to the nested producer, typically through a command-line argument or an environment variable. The nested producer then creates its channel with them:

```swift
// In the nested producer, using identifiers passed by the enclosing producer
let channel = try Channel(
    sessionID: sessionIDFromEnclosingProducer,
    parentID: parentIDFromEnclosingProducer,
    artifactPath: artifactPath
)
```

Where a nested producer writes its data depends on the adopting system's topology, and Structured Results doesn't prescribe it. Two arrangements are supported:

- The nested producer writes to its own artifact, which is later merged (see Merging) with the enclosing producer's artifact.
- The enclosing producer passes a shared artifact location to the nested producer, along with the session identifier and parent event id, and both write into the same artifact.

The adopting system is responsible for managing this, for example based on whether the producers share a filesystem.

#### Lifecycle

Channels have a fixed lifecycle:

- Init (`Channel.init`): when the channel is first created, and once the artifact structure and manifest are in place, it creates its event stream and writes the stream header. The event file is created with exclusive-create semantics: if it already exists, for example because a channel identifier was reused, `init` fails and writes nothing. The channel is not active and must be explicitly started.
- Start (`start()`): the Channel event is the first event a channel sends. Every other event in the channel is sent after it.
- Reporting (`send(event:)` and `save`): while open, the channel sends events in the usual way - spans, instant events, and attachments.
- Finalized (`finalize()`): a channel closes with a Completion event. The Channel event and its Completion form a span, which enables a consumer to compute how long the producer was active.

Before the channel's Completion is recorded, each Attachment event must receive its AttachmentFinalization event, and every span still open in the channel must be closed, so that the channel's Completion is the last event in its stream. A producer can close spans itself, either preemptively (typically with an `interrupted` outcome) or by waiting for the work to complete and recording its outcome. `finalize()` handles whatever remains: it waits for saves still in progress, records a discard finalization for each attachment that was never finalized, and records an `interrupted` Completion for each span still open, before recording the channel's own Completion with the given outcome.

Nested channels are detached from this rule. A channel does not wait for channels nested under it. A nested channel's events and `Completion` may be recorded after the enclosing channel's `Completion`, and a nested channel's state doesn't affect whether the enclosing channel is finished.

The following example records a channel containing a test span with an attachment:

```swift
let channel = try Channel(
    sessionID: "5e551011-4455-4677-8899-aabbccddeeff",
    channelID: "c0ffee00-1b0a-4998-8877-665544332211",
    artifactPath: artifactPath
)
try channel.start()

let test = Event(parentID: "c0ffee00-1b0a-4998-8877-665544332211", payload: Start(message: "testLogin"))
try channel.send(event: test)

let log = Event(parentID: test.id, payload: Attachment(fileName: "console.log"))
try channel.send(event: log)
try await channel.save(attachmentWithID: log.id, bytes: Array(consoleOutput.utf8))

try channel.send(event: Event(parentID: test.id, payload: Completion(outcome: .failed, message: "Login button not found")))
try await channel.finalize()
```

It writes this stream to `c0ffee00-1b0a-4998-8877-665544332211.jsonl`, and the attachment's data to `attachments/9f1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d.log`:

```json
{"ssrStreamVersion":"0.1.0","sessionID":"5e551011-4455-4677-8899-aabbccddeeff","channelID":"c0ffee00-1b0a-4998-8877-665544332211"}
{"id":"c0ffee00-1b0a-4998-8877-665544332211","parentID":"5e551011-4455-4677-8899-aabbccddeeff","timestamp":1757548800.0,"monotonicTime":41234.5,"payloadType":[{"name":"Start","version":"0.1.0"},{"name":"Channel","version":"0.1.0"}],"payload":{"base":{},"processID":24071}}
{"id":"7d3e9a10-2c4b-4f6e-8a1d-5b9c0e3f7a21","parentID":"c0ffee00-1b0a-4998-8877-665544332211","timestamp":1757548800.2,"monotonicTime":41234.7,"payloadType":[{"name":"Start","version":"0.1.0"}],"payload":{"message":"testLogin"}}
{"id":"9f1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d","parentID":"7d3e9a10-2c4b-4f6e-8a1d-5b9c0e3f7a21","timestamp":1757548801.0,"monotonicTime":41235.5,"payloadType":[{"name":"Attachment","version":"0.1.0"}],"payload":{"fileName":"console.log"}}
{"id":"1a2b3c4d-5e6f-4789-9abc-def012345678","parentID":"9f1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d","timestamp":1757548803.1,"monotonicTime":41237.6,"payloadType":[{"name":"AttachmentFinalization","version":"0.1.0"}],"payload":{"size":2048}}
{"id":"2b3c4d5e-6f70-4812-a3b4-c5d6e7f80912","parentID":"7d3e9a10-2c4b-4f6e-8a1d-5b9c0e3f7a21","timestamp":1757548803.2,"monotonicTime":41237.7,"payloadType":[{"name":"Completion","version":"0.1.0"}],"payload":{"outcome":"failed","message":"Login button not found"}}
{"id":"3c4d5e6f-7081-4923-b4c5-d6e7f8091a2b","parentID":"c0ffee00-1b0a-4998-8877-665544332211","timestamp":1757548803.3,"monotonicTime":41237.8,"payloadType":[{"name":"Completion","version":"0.1.0"}],"payload":{"outcome":"succeeded"}}
```

#### Unfinished Channels

A channel without a Completion is unfinished: the producer crashed, was killed, or its stream was truncated. Its events remain valid and should be included in consumer processing, but the consumer can't tell whether the producer reported everything. A stream that is empty or contains only the header is also a valid unfinished stream: the channel was initialized but never started, or the producer crashed before writing the `Channel` event.

An `Attachment` with no `AttachmentFinalization` is interpreted as described under Attachments.

A producer that relaunches after a crash should not reopen the channel used by the previous instance. It should open a new channel, with a new channel ID, in the same session. The parent of the new channel is up to the producer; typically it is the same parent as the interrupted channel it replaces.

#### Multiple Channels in a Session

Channels in a session can run concurrently, on the same machine or on different ones. Each channel writes to its own event file, and attachment files are named for unique event identifiers, so event and attachment files need no synchronization, but artifact creation is the one place that does (see Manifest).

---

### Artifact Layout

This section describes how results data is laid out on disk.

```
$ARTIFACT_PATH/                       Directory
    ssr.json                          Manifest file with format version and session identifiers
    <CHANNEL_ID>.jsonl                Event stream; one file per channel, UUID is channel identifier
    <CHANNEL_ID>.jsonl
    attachments/                      Directory; attachment bytes, keyed by UUID (Attachment's event id)
        <ATTACHMENT_EVENT_ID>.<ext>   If the Attachment fileName had an extension it is appended to the id
        <ATTACHMENT_EVENT_ID>         Attachment with no file extension
```

`$ARTIFACT_PATH` is the directory passed to `Channel.init` as `artifactPath`. It has no required name or extension. The first channel that is instantiated creates the directory, the `attachments/` subdirectory, and the `ssr.json`, as described under Manifest.

#### Version

The layout version at the time of this proposal is `0.1.0`. This is the first version of the layout, and it may evolve in the future.

Layout versions follow SemVer and are compatible according to the rule described under Versioning.

#### Manifest

The manifest file, `ssr.json`, is a single JSON object at the top level of the artifact. Its existence identifies the directory as a Structured Results artifact and it specifies the version for the artifact (which only governs the layout), and lists all of the sessions contained in the artifact. Consumers read the manifest first and a directory without one isn't an artifact. Consumers ignore any unrecognized fields in the manifest, so later layout versions can add fields.

```json
{
  "version": "0.1.0",
  "sessions": ["5e551011-4455-4677-8899-aabbccddeeff"]
}
```

- `version` - Version of the artifact layout, a SemVer version string, as major.minor.patch.
- `sessions` - The identifiers of the sessions the artifact contains. Normally an artifact has results data for one session, but if the artifact is the product of merging, the sessions list will contain the identifier for each merged session. This list has Set semantics - there are no duplicates. An artifact produced by merging multiple sessions is not written to further.

Several channels may be instantiated concurrently against the same `$ARTIFACT_PATH`, so creation of the artifact structure must be atomic:

- The directory and the `attachments/` subdirectory are created with create-if-absent semantics, along with any missing parent directories of `$ARTIFACT_PATH`. A directory that already exists is not an error.
- The manifest is created with exclusive-create semantics: exactly one channel creates it. The manifest must never be observable in a partially written state. A channel that loses the race reads the existing, complete manifest.
- If the manifest already exists, the channel verifies that its `sessions` list contains exactly the channel's own session and no others, and that its `version` is compatible with the layout version the channel writes. Anything else is an error, because a channel must not write to an artifact for a different session or to one that contains multiple sessions.
- Any other failure to create the directory structure or manifest, or to read an existing manifest, is also an error.

An error in any of these conditions must be reported to the caller, and the channel must not continue: it must not write a header or any events.

#### Merging

Merging combines the results data from two or more artifacts into a single artifact. This can be useful for several purposes:

- joining artifacts produced in different locations for the same session, for example if tests are running on multiple devices that each produce their own artifact.
- doing aggregate analysis across multiple sessions, for example to compare the data from a group of recent jobs.
- integrating data from different sessions that are logically related, for example combining data from different CI jobs triggered by the same pull request.

For artifacts to be merged, they must have compatible layout versions (see Version) in their `ssr.json` files. The merging operation does the following:

- A new artifact directory structure is created.
- The `ssr.json` for the new artifact has the highest layout version among the input artifacts and its `sessions` contains all distinct session identifiers from the input artifacts.
- Event stream files and the attachments are all copied into the corresponding locations in the new artifact.
- Event stream files and attachments are not modified in any way.

If two input artifacts contain a file at the same relative path, other than `ssr.json`, which the merge regenerates, the merge is an error and does not produce an artifact. Because event stream and attachment file names are UUIDs, this should not occur in practice.

#### Event Files

Each `<CHANNEL_ID>.jsonl` file holds the stream of one channel, in the format described under Stream Encoding. A channel writes only to its own files - event files are never shared (see Multiple Channels in a Session). Consumers discover event files by scanning the top level of the artifact for files named `<UUID>.jsonl`, with the UUID in lowercase, so producers never have to update a shared file while they run. Consumers ignore other files and directories, apart from `ssr.json` and `attachments/`.

#### Attachment Files

`Attachment` and `AttachmentFinalization` events contain the metadata of an attachment, while the bytes are written as separate files to `attachments/`.

The file name is the `Attachment`'s event identifier, a UUID string, followed by the extension of the `Attachment`'s `fileName`, if it has one. The extension is the suffix after the last period in `fileName`, and it is appended including that period. A `fileName` has no extension if it has no period, if its only period is its first character (a dotfile such as `.gitignore`), or if its last character is a period (such as `foo.`).

#### Summary

Artifacts must conform to the following:

- The manifest must be created before anything is written to an event file.
- The parent of a top-level channel is one of the session identifiers in `sessions`.
- The `sessionID` in each event file's header is one of the session identifiers in `sessions`.
- A channel only writes to an artifact whose manifest lists exactly its own session.
- Channels only write their own event files (one event file per channel).
- Event files are JSON Lines: a header line, then one event per line, each terminated by `\n`.
- Channels cannot be rewritten - producers only append to files and create new ones.

---

## Differences from the Pitch

The Pitch represented the team's investigation and understanding of the problem and approaches to a solution at the time. Since then, the investigation has continued and new information and ideas have been brought from work with other teams and deeper testing of various aspects of the design. This has led to some changes in what is proposed now and we want to share the reasoning for those differences.

### Removal of Transport

In the Pitch, we described a solution into which clients could "plug" different implementations of a layer that would be responsible for capturing and emitting the data, which we called "Transport". A file system transport and an HTTP transport were considered as likely built-in implementations. Clients with specialized needs could implement their own transports to send data over platform-specific API, or other network protocols.

Testing and prototyping were showing us that reliability and performance were significant concerns. Network typically involves buffering, and a producer crash would probably lead to lost data. We looked at write-ahead logging as a guard against this, and it was feasible, but introduced the necessity of external collection to recover lost data. Back pressure on both IPC and network writes were also a concern, with a real likelihood of negative impact on a producer if it had delays while writing.

Writing directly to the file system is the most resilient to faults and least likely to cause bottlenecks or delays. The necessity of separate collection in the case of faults pointed us towards a simplification of the system and a clarification of responsibilities. We may add a collector module in the future that adopters can use for this purpose, but that's not part of our proposal for the first version of Structured Results and it is an intentional design choice to decouple responsibility for network or other transport from the recording module. A pluggable layer would also let third-party transports run inside the producer process, which is the failure mode we wanted to avoid.

Pluggable transport was also a potential way to allow for different serialization technologies - instead of JSON lines, perhaps CBOR, an example specifically part of the original discussion thread. Removing transport does not reflect any intentional limitation around serialization options - we do consider that a distinct potential direction for future expansion, but it's not part of this proposal.

### Refinements

Other differences from the Pitch are refinements of what it described, not changes in direction:

- **A single event envelope.** The Pitch described the stream in general terms, as spans with a start and a finish and leaf records between them, rather than as a specification. This proposal gives that structure a concrete form: every line is an `Event` with the same envelope, spans are expressed through the `Start` and `Completion` payloads, and the tree of events comes from each event's parent.
- **Live results.** Live results remain a goal. The stream is append-only, so a consumer can follow it while it is being written, and a collector can forward it to remote consumers (see Collector in Future Directions).
- **`ssrtool` in the first phase.** The Pitch described a command line tool as a later addition. Toolchain projects remain the core adopters of the libraries, but end users will consume the results those projects produce, so `ssrtool` is part of the first phase.
- **API stability.** The APIs are experimental for now, and will become stable when the stream, layout, and built-in payload versions reach `1.0.0` together.

---

## Future Directions

This proposal is the first of several that cover the core design of Structured Results. The following topics are intentionally not addressed here and are left to future proposals or to the implementation, as described in About This Proposal.

### Related Designs

- Reader and CLI. The design of the `StructuredResultsReader` module, including the type registry used for type resolution, and of `ssrtool`, including aggregation and analysis, will be proposed in a separate document.
- Domain payloads. Payloads that model specific categories of results data, such as build and test activities, will be proposed separately.
- JSON Schemas. The schema documents for the stream header, events, predefined payloads, and manifest are not included in this proposal.
- Other serializations. JSON Lines is the only serialization of the event stream. The design doesn't preclude alternatives.
- Collector. Delivering results to destinations other than the local file system is the job of a separate collector, which may be proposed as a module in the future. We expect a collector to tail the event files and send their contents on, and to upload attachments as they are written, possibly filtering the tailed event stream for `AttachmentFinalization` events that record a `size`. A collector should not treat the appearance of a file in `attachments/` as a sign that the attachment is complete, since a file can be present while it is still being written, or before the channel removes it after a failed save. An attachment's data is complete only once its `AttachmentFinalization` records a `size`.

### Recorder API

- Errors. The errors thrown by `Channel` and how they are categorized are not specified.
- Validation. Which rules of the format the Recorder enforces when events are recorded, such as a parent existing in the same stream, at most one `Completion` per span, or the format of identifiers, is not specified.
- Messages for the channel's span. `start()` and `finalize()` don't accept a message for the channel's `Start` and `Completion`.
- Incremental attachment writes. Attachment data is saved in a single call. An API for writing large attachments, such as recordings, incrementally may be added.

### Format

- Consumer handling of invalid content. How consumers treat content that violates the format, such as malformed lines, events whose parent can't be found, events after a channel's `Completion`, or unsupported stream and layout versions, is not specified.
- Meaning of a channel's outcome. The outcome of a channel's `Completion` isn't defined in relation to the outcomes of the work it reported.
- Producer and session metadata. The `Channel` payload doesn't identify the producer beyond its process ID, for example by name, version, or host, and sessions have no metadata of their own.
- Attachment metadata. Attachments have no media type or content digest; consumers can only infer the type of data from the `fileName` extension.
- Referencing existing attachments. An affordance for referring to a previously created attachment from multiple places in the stream will be part of a future proposal.
- Cross-session references. A mechanism for referring to events in other sessions may be proposed in the future. It would use a separate affordance, not an event's `parentID`. An example use case is an incremental build that refers to the build tasks, executed in an earlier session, whose products it reuses.

### Artifacts

- Merging details. Which module implements merging, whether identical files present in more than one input are permitted, and how a merge treats inputs that are still being written are not specified.
