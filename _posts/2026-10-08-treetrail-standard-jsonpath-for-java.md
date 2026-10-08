---
title: "Treetrail: JSONPath for Java that answers like everyone else"
description: "Treetrail 0.2.0 implements RFC 9535, the JSONPath standard, for Java: the same results as standard implementations in other languages, safe with untrusted queries, adapters for Jackson, Gson and JSON-P, and test matchers for AssertJ and Spring."
date: 2026-10-08
tags: [java, jsonpath, json, rfc9535, spring, testing]
---

JSONPath is everywhere in Java: in Spring's `MockMvc` assertions, in API gateways, in configuration
that picks values out of webhooks. For most of its life it had no specification, so every library
decided for itself what `$..book[-1:]` or `$[?(@.price < 10)]` should return. In February 2024 the
IETF published [RFC 9535](https://www.rfc-editor.org/rfc/rfc9535), the JSONPath standard, together with a
[Compliance Test Suite](https://github.com/jsonpath-standard/jsonpath-compliance-test-suite) of 706 cases.

[Treetrail](https://github.com/treetrail/treetrail) is a Java implementation of that standard. Version
0.2.0 is out on Maven Central today, and it is the first one I'm announcing.

## What it is for

**The same answer in every language.** A query that runs in your Java service, in a Python script
and in a TypeScript frontend should select the same nodes. Treetrail passes all 706 cases of the
Compliance Test Suite on Java 17, 21 and 25, and a single failing case fails the build.

**One kind of result.** Every query returns a list of nodes, whether it selects one value or many,
and every node carries its normalized path. No configuration option changes what a query returns.

```java
JsonPath path = JsonPath.compile("$.store.book[?@.price < 10].title");

NodeList<Object> nodes = path.query(document);
nodes.values(); // ["Sayings of the Century", "Moby Dick"]
nodes.paths();  // ["$['store']['book'][0]['title']", "$['store']['book'][2]['title']"]
```

**The JSON tree you already have.** The core has no dependencies and works on plain Java maps and
lists, or parses JSON text itself. Adapters query Jackson 2 and 3, Gson and Jakarta JSON-P trees
directly and return the library's own nodes, so you can keep working with them.

```java
JsonNode document = objectMapper.readTree(json);
NodeList<JsonNode> books = JsonPath.compile("$.store.book[?@.price < 10]")
        .query(document, Jackson2Model.INSTANCE);
```

**Queries you don't control.** If users, tenants or documents supply the queries, a query must not
be able to stall the service. Regular expressions in `match()` and `search()` run on an automaton
without backtracking, in time linear in the input. `((a+)+)+b` against 24 `a`s and a `!` keeps
`java.util.regex` busy for 0.4 seconds, twice as long with every further `a`; Treetrail needs less
than a tenth of a microsecond. Every run has a budget of visited nodes and a depth limit, and
0.2.0 caps the memory that cached regex automata may hold at about 10 MB, however many different
patterns a document brings along.

## New in 0.2.0: test assertions

Most JSONPath in Java code bases lives in tests. Two new modules bring the standard there:

```java
// jsonpath-assertj
assertThatJson(body).jsonPath("$.store.book[?@.price < 10].title").containsExactly("Sayings of the Century", "Moby Dick");

// jsonpath-spring-test, for MockMvc and WebTestClient
mockMvc.perform(get("/store"))
        .andExpect(jsonPath("$.store.book[?@.price < 10].title").values("Sayings of the Century", "Moby Dick"));
```

Values compare as JSON values, so `399`, `399L` and `399.0` all match the JSON number `399`. A failing
assertion names the expression and shows the values it found, with their paths.

## Coming from Jayway JsonPath

[Jayway JsonPath](https://github.com/json-path/JsonPath) has served the Java world since 2011, long
before there was a standard, and it is the library Spring's `jsonPath(...)` matchers use. Its syntax
and results differ from RFC 9535 in places: filters need parentheses, functions like `.length()`
come at the end of a path, and some selectors answer differently. On the array `[0, 1, …, 9]`, for
example:

| Query | RFC 9535 | Jayway JsonPath 3.0.0 |
| --- | --- | --- |
| `$[1:6:2]` | `[1, 3, 5]` | `[1, 2, 3, 4, 5]` |
| `$[9:0:-1]` | `[9, 8, 7, 6, 5, 4, 3, 2, 1]` | `[]` |

So switching should not be a leap of faith. Treetrail brings two tools for it:

- `jsonpath-migration` runs your expressions with Jayway JsonPath and with Treetrail against sample
  documents and reports every difference, with a rewrite hint for Jayway-only syntax.
- `jsonpath-rewrite` contains an [OpenRewrite](https://docs.openrewrite.org) recipe that finds every
  JSONPath expression passed to Jayway or to Spring's matchers and assesses each one.

The README has a [table of Jayway idioms and their equivalents](https://github.com/treetrail/treetrail#jayway-idioms-and-their-equivalents),
and [a case-by-case comparison](https://github.com/treetrail/treetrail/blob/main/docs/jayway-vs-rfc9535.md)
with the Compliance Test Suite.

## How it is tested

The compliance suite covers the grammar and the semantics, but not what happens with inputs nobody
thought of. So besides the suite, which runs against every JSON model:

- **Fuzzing** with [Jazzer](https://github.com/CodeIntelligenceTesting/jazzer) in every CI build and
  every night: compiling, querying, matching regular expressions and parsing JSON may only throw the
  documented exceptions and must finish each input within five seconds. It found one bug before this
  release: a regular expression that repeated an empty group more than a billion times took 46 seconds to compile.
- **Property tests** with [jqwik](https://jqwik.net), for example that every normalized path selects
  exactly its node.
- **Differential tests:** 20,000 random regular expressions against `java.util.regex`, and 2,000
  random queries against [jsonpath-rfc9535](https://github.com/jg-rp/python-jsonpath-rfc9535), a
  Python implementation of the standard. Where the two disagreed, I checked the RFC. Treetrail was
  right each time, and the reference had six small bugs, which I've
  [reported upstream](https://github.com/jg-rp/python-jsonpath-rfc9535/issues?q=is%3Aissue%20author%3Achristoph-sens).
  Having a careful second implementation to test against was a great help.

## Try it

```kotlin
dependencies {
    implementation("io.github.treetrail:jsonpath-core:0.2.0")
}
```

Java 17 or later, Apache License 2.0. Every release is signed, ships an SBOM per module and has a
build provenance attestation. The API may still change before 1.0, which is a good reason to tell me
now what doesn't fit your use case: [open an issue](https://github.com/treetrail/treetrail/issues).
