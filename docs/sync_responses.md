Synchronous Responses
=====================

EXPERIMENTAL: The features outlined in this document are considered experimental
and are therefore subject to change outside of major version releases.

In a regular Benthos stream pipeline messages flow in one direction and
acknowledgements flow in the other:

``` text
    ----------- Message ------------->

Input (AMQP) -> Processors -> Output (AMQP)

    <------- Acknowledgement ---------
```

For most Benthos input and output targets this is the only workflow available.
However, Benthos has support for a number of protocols where this limitation is
not the case.

For example, HTTP is a request/response protocol, and so our `http_server` input
is capable of returning a response payload after consuming a message from a
request.

When using these protocols it's possible to configure Benthos stream pipelines
that allow messages to pass in the opposite direction, resulting in response
messages at the input level:

``` text
           --------- Request Body -------->

Input (HTTP Server) -> Processors -> Output (Sync Response)

           <------- Response Body ---------
```

## Routing Processed Messages Back

It's possible to route the result of any Benthos processing pipeline directly
back to an input with a [`sync_response`][sync-res] output:

``` yaml
input:
  http_server:
    path: /post
pipeline:
  processors:
  - text:
      operator: to_upper
output:
  type: sync_response
```

Using the above example, sending a request 'foo bar' to the path `/post` returns
the response 'FOO BAR'.

It's also possible to combine a `sync_response` output with other outputs using
a [`broker`][output-broker]:

``` yaml
input:
  http_server:
    path: /post
output:
  broker:
    pattern: fan_out
    outputs:
    - kafka:
        addresses: [ TODO:9092 ]
        topic: foo_topic
    - type: sync_response
      processors:
      - text:
          operator: to_upper
```

Using the above example, sending a request 'foo bar' to the path `/post` passes
the message unchanged to the Kafka topic `foo_topic` and also returns the
response 'FOO BAR'.

NOTE: It's safe to use these mechanisms even when combining multiple inputs with
a broker, a response payload will always be routed back to the original source
of the message.

## Routing Output Responses Back

Some outputs, such as [`http_client`][http-client-output], have the potential to
propagate payloads received from their destination after sending a message back
to the input:

``` yaml
input:
  http_server:
    path: /post
output:
  http_client:
    url: http://localhost:4196/post
    verb: POST
    propagate_result: true
```

With the above example a message received from the endpoint `/post` would be
sent unchanged to the address `http://localhost:4196/post`, and then the
response from that request would get returned back. This basically turns Benthos
into a proxy server with the potential to mutate payloads between requests.

The following config turns Benthos into an HTTP proxy server that also sends all
request payloads to a Kafka topic:

``` yaml
input:
  http_server:
    path: /post
output:
  broker:
    pattern: fan_out
    outputs:
    - kafka:
        addresses: [ TODO:9092 ]
        topic: foo_topic
    - http_client:
        url: http://localhost:4196/post
        verb: POST
        propagate_result: true
```

[sync-res]: ./outputs/README.md#sync_response
[http-client-output]: ./outputs/README.md#http_client
[output-broker]: ./outputs/README.md#broker