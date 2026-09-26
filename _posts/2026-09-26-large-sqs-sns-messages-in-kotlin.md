---
title: "Large SQS and SNS messages in Kotlin: the extended client pattern without Java baggage"
description: "The AWS Java extended client libraries for SQS and SNS pin old Jackson versions with known advisories. sqsoverflow and snsoverflow port them to aws-sdk-kotlin and coroutines, without Jackson."
date: 2026-09-26
tags: [kotlin, aws, sqs, sns, s3, coroutines]
image: /assets/images/overflow-social-preview.png
---

Every SQS queue and SNS topic has a message size limit. When a payload is larger, the usual answer
is the *claim check* pattern: put the payload in S3, send a small pointer instead, and resolve the
pointer on the receiving side. AWS ships this pattern as two Java libraries,
[amazon-sqs-java-extended-client-lib](https://github.com/awslabs/amazon-sqs-java-extended-client-lib)
and [amazon-sns-java-extended-client-lib](https://github.com/awslabs/amazon-sns-java-extended-client-lib),
both built on [payload-offloading-java-common-lib-for-aws](https://github.com/awslabs/payload-offloading-java-common-lib-for-aws).

If your service is written in Kotlin on [aws-sdk-kotlin](https://github.com/awslabs/aws-sdk-kotlin),
those libraries don't fit: they wrap the AWS SDK for Java v2 clients, not the Kotlin ones. So I
ported them. The result is three small libraries on Maven Central:

- [s3overflow](https://github.com/christoph-sens/s3overflow): the payload store (S3 upload, pointer, download, delete)
- [sqsoverflow](https://github.com/christoph-sens/sqsoverflow): an `SqsClient` that offloads large message bodies
- [snsoverflow](https://github.com/christoph-sens/snsoverflow): an `SnsClient` that offloads large message bodies

This post covers why the Java libraries are a liability today, how the ports work, and what you
need to know before migrating.

## Where the Java libraries stand today

The extended clients do their job, but their foundation has aged:

- **Old dependencies with known advisories.** Both clients depend on
  `payloadoffloading-common` 2.2.0, whose last release was in March 2024. It pins
  `jackson-databind` 2.15.2 and `jackson-core` 2.16.0. As of September 2026, the GitHub Advisory
  Database lists seven advisories affecting exactly these versions, three of them rated high,
  among them bypasses of Jackson's polymorphic type validation and a denial-of-service issue in the
  async parser. Even the latest SQS client release (2.1.3, July 2026) still pulls in these versions
  through the common library.
- **Java 8 bytecode and a pre-coroutine design.** All three libraries compile for Java 8 and keep
  separate sync and async client classes with largely duplicated pass-through code.
- **Slow release cadence.** The SNS client's latest release (2.1.0) is from March 2024, the common
  library's from the same month.

To be fair: whether an advisory is exploitable depends on how Jackson is used, and the extended
clients only (de)serialize a small pointer object. You can also override the Jackson versions in
your own build. But every dependency scanner flags these versions, someone has to triage the
findings again and again, and forcing newer Jackson versions under a library that was never tested
against them is its own risk. Removing Jackson from this code path makes the question disappear.

## What the size limits are today

The limits moved recently, which changes when offloading is needed at all:

- **SQS** accepts messages up to **1 MiB** (body plus attributes). Until August 2025 the limit was 256 KiB.
- **SNS** accepts **256 KiB** by default. Since September 2026 a topic can accept up to **1 MiB**
  when you raise its `MaximumMessageSize` attribute.

sqsoverflow therefore offloads at 1 MiB by default, snsoverflow at 256 KiB. Both are configurable
through `payloadSizeThreshold`.

## Usage

```kotlin
dependencies {
    implementation("com.christoph-sens:sqsoverflow:1.1.0")
}
```

```kotlin
val s3Client = S3Client.fromEnvironment { region = "eu-central-1" }
val sqsClient = SqsClient.fromEnvironment { region = "eu-central-1" }

val client = SqsExtendedClient(
    sqsClient,
    SqsExtendedClientConfig(payloadStore = S3BackedPayloadStore(s3Client, bucketName = "my-payload-bucket")),
)

client.sendMessage(SendMessageRequest { queueUrl = myQueueUrl; messageBody = largePayload })

val messages = client.receiveMessage(ReceiveMessageRequest { queueUrl = myQueueUrl }).messages.orEmpty()
for (message in messages) {
    process(message.body) // the original payload, already resolved from S3
    client.deleteMessage(DeleteMessageRequest { queueUrl = myQueueUrl; receiptHandle = message.receiptHandle })
}
```

`SqsExtendedClient` *is* an `SqsClient`, so you can pass it anywhere a plain client is expected.
Messages under the threshold go straight to SQS. Larger ones are written to S3, and SQS only carries
a small JSON pointer plus the `ExtendedPayloadSize` message attribute. On receive, the client
fetches the payload and hands you the original body. On delete, it removes the S3 object as well
(`cleanupS3Payload`, on by default).

snsoverflow works the same way for `publish` and `publishBatch`. SNS has no receive side, so the
subscriber resolves the pointer: for an SNS topic that fans out to an SQS queue with raw message
delivery, that's sqsoverflow's `SqsExtendedClient`.

## Why a port instead of a wrapper around the Java library

### One suspend API instead of sync and async classes

The Java originals come in two flavors each, for example `AmazonSQSExtendedClient` for `SqsClient`
and `AmazonSQSExtendedAsyncClient` for `SqsAsyncClient`. aws-sdk-kotlin clients are `suspend`-based
from the start, so one class covers both cases and fits naturally into coroutine code.

### Interface delegation replaces a thousand lines of pass-through code

Most of the SQS interface has nothing to do with payload offloading: `createQueue`, `listQueues`,
`tagQueue` and dozens more. The Java library forwards each of these by hand in a base class of
roughly 1,150 lines. Kotlin's interface delegation does this in one line:

```kotlin
class SqsExtendedClient(
    private val sqsClient: SqsClient,
    private val clientConfig: SqsExtendedClientConfig,
) : SqsClient by sqsClient {
    override suspend fun sendMessage(input: SendMessageRequest): SendMessageResponse { /* offload if needed */ }
    // ... only the methods with real offload logic are overridden
}
```

sqsoverflow overrides eight methods: `sendMessage`, `sendMessageBatch`, `receiveMessage`,
`deleteMessage`, `deleteMessageBatch`, `changeMessageVisibility`, `changeMessageVisibilityBatch` and
`purgeQueue`. snsoverflow overrides two: `publish` and `publishBatch`. Everything else is delegated
to the wrapped client. All three libraries together are about 730 lines of Kotlin, license headers
included.

### No Jackson dependency

The Java libraries serialize the S3 pointer with Jackson. The ports use kotlinx.serialization, so
nothing in their runtime classpath is Jackson, and the advisories described above do not apply to
them. Fewer transitive dependencies means fewer libraries to keep patched. Dependabot keeps the
remaining dependencies current, and CodeQL and dependency review run on every pull request in all
three repositories.

### Less configuration surface

The original configuration classes carry options for client-side encryption strategies and canned
ACLs. The ports take a `PayloadStore` and a handful of behavior flags (`payloadSizeThreshold`,
`alwaysThroughS3`, `cleanupS3Payload`, `ignorePayloadNotFound`, `s3KeyPrefix`). Encryption is
configured where it belongs: on the bucket, with SSE-S3 or SSE-KMS.

`publishBatch` in snsoverflow is offload-aware as well. The Java SNS library predates the SNS
`PublishBatch` API and only handles `publish`.

## Migrating from the Java libraries

The important caveat first: **the ports are not wire-compatible with the Java libraries.** The
pointer JSON and the receipt-handle format differ. A message sent by the Java extended client cannot
be resolved by sqsoverflow, and vice versa. Switch all producers and consumers of a queue at the
same time, or drain the queue first.

The `ExtendedPayloadSize` attribute name is the same, so SNS-to-SQS fan-out works between
snsoverflow and sqsoverflow.

Before (Java):

```java
ExtendedClientConfiguration config = new ExtendedClientConfiguration()
    .withPayloadSupportEnabled(s3Client, "my-payload-bucket");
SqsClient client = new AmazonSQSExtendedClient(SqsClient.builder().build(), config);
```

After (Kotlin):

```kotlin
val client = SqsExtendedClient(
    SqsClient.fromEnvironment(),
    SqsExtendedClientConfig(payloadStore = S3BackedPayloadStore(s3Client, bucketName = "my-payload-bucket")),
)
```

Options without an equivalent:

| Java option | In the ports |
|---|---|
| Client-side encryption (`ServerSideEncryptionStrategy`) | Configure SSE-S3 or SSE-KMS on the bucket |
| `ObjectCannedACL` | Use bucket policies |
| SNS per-message `"S3Key"` attribute | Use `s3KeyPrefix` in the config |
| Legacy `SQSLargePayloadSize` attribute name | Always `ExtendedPayloadSize` |

## Verifying what you download

Every release is built and published by a GitHub Actions workflow. The jars, POM and Gradle module
metadata on Maven Central carry a signed
[build provenance attestation](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations)
that ties each file to the repository, the workflow and the tagged commit it was built from. You can
check a downloaded jar with the GitHub CLI:

```bash
gh attestation verify sqsoverflow-1.1.0.jar --repo christoph-sens/sqsoverflow
```

## Licensing

sqsoverflow and snsoverflow are derivative works of the AWS libraries under Apache-2.0, with
per-file attribution and a NOTICE file listing what was changed. s3overflow is an independent
reimplementation that takes no code from `payload-offloading-java-common-lib-for-aws`.

## Try it

- s3overflow: <https://github.com/christoph-sens/s3overflow>
- sqsoverflow: <https://github.com/christoph-sens/sqsoverflow>
- snsoverflow: <https://github.com/christoph-sens/snsoverflow>

Issues and pull requests are welcome. If you run into a case the libraries don't cover yet, such as
another AWS service that could use the same pattern, open an issue.

*I'm a freelance developer and consultant working on Kotlin, Java and AWS. The story of how these
libraries came about is on my website (in German):
[1150 Zeilen Legacy, eine Zeile Kotlin](https://www.christoph-sens.com/post/der-scan-befund-der-nie-verschwindet-wie-eine-ki-drei-aws-legacy-bibliotheken-an-einem-nachmittag).*
