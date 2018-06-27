# Taskhawk

TaskHawk is a simple asynchronous task queue that works on AWS SQS. It's reasonable to think of
this as a replacement for Celery (Python) for people who do not need the complexity of celery, or
want to deal with failures in Celery.

## Why?

Like a lot of other Python developers, when we needed asynchronous worker pattern for our project,
we decided to go with Celery. Since Celery did not have reasonable support any queue systems other
than RabbitMQ, that was the our choice of backend. It worked well to start with, however as we
deployed it into production and scaled the number of messages flowing through the
system, we started to see a bunch of odd failures, such as, [worker hangs](https://github.com/celery/celery/issues?utf8=%E2%9C%93&q=label%3A%22Worker+Hangs%22),
[broken logging](https://github.com/celery/celery/issues/4326), [RabbitMQ issues](https://github.com/celery/celery/issues?utf8=%E2%9C%93&q=is%3Aissue+label%3A%22Component%3A+RabbitMQ+Broker%22).

With Celery v4, came the promise of AWS SQS, which we love and use in other places, so we attempted
a switch from RabbitMQ to SQS but found that the support was mostly terrible, including [exponentially
growing duplicate messages!](https://github.com/celery/celery/issues/4002#issuecomment-327255816).

Having used celery in a bunch of projects, we realized we did not use or need a majority of functionality
offered by celery, such as a results backend, task monitoring, chords, groups, etc. All we needed
was a simple and reliable queue that would invoke worker code asynchronously.

Automatic relies heavily of AWS, including Simple Queue Service. While Amazon does not provide any
SLA guarantees, in practice SQS scales infinitely with minimal data loss. SQS also provides at least
once delivery guarantee. We prefer using AWS infrastructure over self-managed infrastructure wherever
the offering matches our needs. We call this "exporting our nines (9s) to Amazon".

After all the problems trying to production-ize our Celery usage, we decided to write our own
async task queue that works on AWS SQS. Since SQS gives us reliability and durability, we
do not have to worry about the queue itself. Also having excellent API support and libraries like
boto, we do not have to worry about the SQS interface.

## What?

Taskhawk is a simple library to work with SNS/SQS to dispatch and consume asynchronous tasks. It supports
apps running on AWS Lambda as well as traditional apps. Your code need not run in AWS, it just needs
to be able to talk to AWS APIs. For Lambda apps, Taskhawk uses SNS to dispatch tasks which can execute
your lambda function. All SQS messages need to be ack-ed after successful processing. Tasks are held in
SQS queue (or internal Lambda queue) until they’re successfully executed, or until they fail a configurable
number of times. Failed tasks are moved to a Dead Letter Queue, where they’re held for 14 days,
and may be examined for further debugging.

Taskhawk provides 4 queues which may be used for independent monitoring and scaling needs. These queues
are divided by priority - default, high priority, low priority, and batch priority. In
addition to on-demand task dispatch, Taskhawk also provides a cron mechanism for invoking tasks at
regular intervals, or on a fixed schedule. This is implemented using [Cloudwatch Rules](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html),
and is the equivalent for celery beat.

## Provisioning

Taskhawk requires an SQS queue or an SNS topic (for Lambda apps), and credentials to access this resource.
It's recommended that the infra be set up using [Terraform](https://www.terraform.io/), with provided [modules](https://registry.terraform.io/search?q=taskhawk&verified=false).
There is a [CLI tool](https://github.com/Automatic/taskhawk-terraform-generator) available to manage
the Terraform config so you do not have to deal with Terraform syntax.

## Protocol & limits

[Relevant SQS limits](https://aws.amazon.com/sqs/faqs/#limits-restrictions) -

- Message size: 256 KB
- Message retention: 14 days

## Spec

This section describes the spec for any Taskhawk library.

### Message Format

Message format is versioned using semver, and current version (`v1.0`) is as follows:

```
{
    // unique message identifier uuid v4 for every
    // instance of a new message
    "id": "b1328174-a21c-43d3-b303-964dfcc76efc",

    "metadata": {
        // taskhawk priority
        "priority": "high",

        // message creation timestamp. Either
        // epoch milli-seconds, or ISO 8601 string
        "timestamp": 1460868253255,

        // format version for message payload
        "version": "1.0"
    },

    // user defined arbitrary headers (string values)
    "headers": {
        ...
    },

    // name of the task
    "task": "tasks.send_email",

    // required args (optional). May be any valid 
    // JSON value.
    "args": [
        "email@automatic.com",
        "Hello!"
    ],

    // optional args (optional). May be any valid
    // JSON value.
    "kwargs": {
        "from_email": "spam@example.com"
    }
}
```

For a list of all versions, see [message formats](message-formats/).

### Library Implementation

The library should allow publishing to a configurable Taskhawk queue/topic with provided IAM credentials.
Message should be published as a plain text JSON with the format as described above. A new random
message id must be generated when message is initialized, and must be preserved during retries if
attempted. All key-value pairs in `headers` must be published as SQS/SNS message attributes.

Optionally, libraries may provide hooks for modifying message contents/headers before publishing,
or for inspecting message contents/headers after consuming before processing the message. This is useful,
for example, to inject a request id for cross-application tracing or general debugging.

#### SQS

Consumer should fetch SQS message attributes on de-queue but NOT use it to modify the message payload.
Consumer should only ack a message after successful processing. Consumer should allow for clean
and quick termination that may be used to shut down the app gracefully.

#### Lambda

Consumer is invoked by AWS with a set of SNS messages. On failure on any of the messages, an exception
must be raised to indicate failure to AWS. Consumer should allow quick termination that may be used to shut
down the app gracefully.

## Libraries

Taskhawk provides and supports the following implementations:

- [Python](http://taskhawk.readthedocs.io/en/latest/)
- [Golang](https://godoc.org/github.com/Automatic/taskhawk-go)

## Roadmap

- Support for other queueing systems such as Google Task Queue.
- Monitoring tools for keeping track of queue health.

## Contributors

Project Leads
  - [Aniruddha Maru](https://github.com/maroux/)
  - [Michael Ngo](https://github.com/mngo87)

Committers
  - [Eytan Hanig](https://github.com/eytanhanig/)
  - [Luke Culbertson](https://github.com/lucasnad27)
  - [Jonathon Klobucar](https://github.com/klobucar)
  - [Manish Singh](https://github.com/yosh)
  - [Marcin Bachry](https://github.com/mbachry/)
  - [Ryan Chong](https://github.com/chongr)
