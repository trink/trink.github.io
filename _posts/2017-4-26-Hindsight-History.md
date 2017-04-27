---
layout: post
title: The Brief History/Origin of Hindsight
---

Although the open source version of Hindsight is relatively new, its predecessor
was developed last decade at Tellme/Microsoft to handle all the IVR (Interactive
Voice Response) call logs of some of the top fortune 500 companies (American
Airlines, UPS, FedEX, American Express, Fidelity to name a few). The system went
into production in 2010 and was expanded that year to also process the global
Bing voice search logs. At that point in time the system was handling billions
of log messages a day accounting for ~1.5TB of data (in a proprietary binary log
format). A single Hindsight box could accommodate all of the data for real time
processing; that included ~60K concurrent analysis sandboxes (three for each
toll free number we were servicing) monitoring things like voice recognition
accuracy, call volumes/duration/latencies, and SLAs in addition to the actual
service components and hardware utilization. However, the actual production
setup consisted of two boxes per data center running in parallel; this setup
provided a hot fail over and a mechanism to easily roll-out/test new releases in
production. The purpose of the sandbox design was to allow each customer to run
their own business logic against data relevant to their application and provide
a self-serve monitoring portal in addition to what we already deliver. The
system remained in this configuration until the Solaris based infrastructure was
decommissioned a year or two after I left the company.

## The Role of Heka

When I started at Mozilla the Heka project was already underway and the sandbox
design was a good fit for the self-serve data analysis requirement. The original
sandbox was implemented in C++ but it was not open source so I re-wrote it in C
to interface with CGO. As the project evolved the sandbox code was pulled out
into a separate repository, so it could be re-used, Heka was
[deprecated](https://mail.mozilla.org/pipermail/heka/2016-May/001059.html) and
then replaced by Hindsight*. Hindsight supports the Heka sandbox API (which is
why it is still called a Heka sandbox) meaning that 99% of existing sandboxes
could be run with no modification and enjoy a 1.7x increase in speed due to the
removal of the CGO interface. The redesign also produced a system that is an
order of magnitude smaller (memory utilization, code size and executable size)
and an order of magnitude faster in terms of
[throughput](https://github.com/mozilla-services/hindsight/blob/master/docs/performance.md)
while providing at least once delivery guarantees and better reliability in our
production environment.

## The Evolution of Hindsight from Tellme to Mozilla

One of the biggest changes to the infrastructure was the addition of
input/output plugins (the term plugin is used here since they are no longer
isolated from the system but over time the terms plugin and sandbox are now used
interchangeably). At Tellme the logging infrastructure was based on the Tellme
Binlog and log servers. This meant all inputs and outputs were standardize and
didn't require any plugins; they were simply built into the system. However,
since this version of Hindsight is designed as a generalized data stream
processor the inputs and outputs need to interact with many different transports
and perform much more data transformation. Heka contributed to the input/output
plugin design through the concepts of decoders and encoders to separate the data
transport from the transformation. The data transformation power of the I/O
plugins is what really sets them apart from other systems due to incredible
modules like [lpeg](http://www.inf.puc-rio.br/~roberto/lpeg/lpeg.html),
[rjson](https://mozilla-services.github.io/lua_sandbox_extensions/rjson/index.html)
and
[parquet](https://mozilla-services.github.io/lua_sandbox_extensions/parquet/index.html).

## Hindsight at Mozilla

As far as I know we are running the largest data set through Hindsight at the
moment averaging around 500MM documents a day. I say documents instead of logs
because they are the [JSON telemetry
data](https://gecko.readthedocs.io/en/latest/toolkit/components/telemetry/telemetry/data/index.html)
we collect from the Firefox browsers ranging in size from a few KiB per document
to around a MiB totaling around 6TB a day.  A handful of Hindsight machines
perform data collection, JSON parsing, JSON schema validation, data
transformation, real time analytics and S3 data warehouse loading all tied
together by a Kafka cluster. The specifics of what we are doing with Hindsight
will be covered in future posts but I was hoping a little history would be a
good place to start.

\* The question why didn't you look at Rust instead of C has been asked many
times. Short answer, we did look at Rust in August 2014 before choosing C; long
answer: a different post.
