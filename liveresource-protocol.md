# LiveResource Protocol

LiveResource is a protocol for receiving live updates of web resources. Resources may be of any kind, for example REST API endpoints or files. The protocol aims to to be simple, minimal, and align with HTTP/REST conventions.

## Design goals

* Given a resource URL, it should be possible for a receiving entity to listen for updates of the resource at that URL.
* It should be possible to deliver updates instantly, without the recipient needing to poll.
* It should be possible to listen for three types of updates:
    1. The entire content of a resource.
    2. Only the content of a resource that has changed.
    3. An indication that a resource has changed without necessarily providing any of its content (aka an update "hint").
* HTTP concepts should be reused where possible, for example ETags for resource versioning.
* HTTP should be preferred as the data transport where possible, but alternative protocols such as WebSockets may be used to work around HTTP limitations.
* The protocol should be able to work efficiently with HTTP/2.
* Resources should be able to use any content type, including binary content. Put another way, resource content should not be limited to JSON. This ensures the protocol remains future-proof.
* Servers should have an incremental upgrade path, so that some benefits of the protocol can be realized without needing to implement it in its entirety.
* Simple servers, complex clients. Related to the previous point, client libraries should try to support as much of the protocol as makes sense for the target platform, to ensure compatiblity with a variety of servers that may only implement specific parts.

## Overview

Resources advertise support for live updates by including certain HTTP response headers.

For example, suppose a client requests a resource:

{% highlight http %}
GET /resource HTTP/1.1
{% endhighlight %}

The response from the server may include `Link` headers indicating various live update mechanisms:

{% highlight http %}
HTTP/1.1 200 OK
Link: </resource>; rel="value-wait value-stream"
Link: </resource/subscription/>; rel=value-callback
...
{% endhighlight %}

This information can also be obtained by making a HEAD request, if the client wants to see what a resource supports without retrieving its content.

The client can then listen for updates by making use of the advertised mechanisms.

## Update types

There are three update types:

* Value: The entire value of a resource, headers and content.
* Changes: Data indicating the changed parts of a resource. The LiveResource protocol does not define a specific format for changes. The server may use any data format to represent changes of a resource. For example, it could return a subset of items in the case of an updated collection, or a text diff, or a JSON patch to show the changes of a JSON object, etc. It is up to clients to know how to interpret such payloads.
* Hint: An indication that the resource has changed without providing any of resource data. When a client receives a hint, it will need to make a separate GET request to retrieve the data.

## Update mechanisms

There are five update mechanisms:

* Plain-polling: The client polls the resource on an interval to retrieve the latest data.
* Long-polling: The client makes a request and the server waits until the resource has changed before responding.
* Stream: The client listens for updates using the Server-Sent Events protocol.
* Socket: The client listens for updates using WebSockets.
* Callback: The receiver registers a callback URL that the server should make requests to when the resource has changed.

## Link types

`Link` headers are used to indicate live update mechanisms. Given the various update types and update mechanisms, the following link relations are defined based on the possible combinations:

Link rel           | Meaning
------------------ | ----------------------------------------------------------------------------------------------------
`value-wait`       | Long-polling updates of a resource's entire value
`value-stream`     | Server-sent events (SSE) updates of a resource's entire value
`value-callback`   | Base URI for subscription management of HTTP callback (webhook) updates of a resource's entire value
`changes`          | Immediate updates of the changes of a resource
`changes-wait`     | Long-polling updates of the changes of a resource
`changes-stream`   | Server-sent events (SSE) updates of the changes of a resource
`changes-callback` | Base URI for subscription management of HTTP callback (webhook) updates of the changes of a resource
`hint`             | Immediate check to see if a resource has changed
`hint-wait`        | Long-polling updates of hints that a resource has changed
`hint-stream`      | Server-sent events (SSE) updates of hints that a resource has changed
`hint-callback`    | Base URI for subscription management of HTTP callback (webhook) updates of hints that a resource has changed

The `changes` and `hint` link types are not live update mechanisms, but are used to indicate pull mechanisms for update types which cannot be assumed to exist but are needed in certain contexts (e.g. recovery after connection loss, plain polling, or use with `multiplex-socket` as described below). There is no `value` link type since it is implied that all resources support this by making a GET request on the original resource URL.

Additionally, there are link types for multiplexing:

Link rel           | Meaning
------------------ | ----------------------------------------------------------------------------------------------------
`multiplex-wait`   | Base URI for accessing multiple resources in a single request, with support for long-polling
`multiplex-socket` | Base URI for accessing multiple resources via a single WebSocket connection

Some link relations imply others:

Link rel           | Implies
------------------ | -----------
`changes-wait`     | `changes`
`hint-wait`        | `hint`

## Update protocols

### Long-polling

Resources may announce support for updates via long-polling by including an `ETag` and appropriate `Link` header:

{% highlight http %}
GET /object HTTP/1.1
...

HTTP/1.1 200 OK
ETag: "b1946ac9"
Link: </object>; rel=value-wait
...
{% endhighlight %}

To make use of the `value-wait` link, the client issues a GET to the linked URI with the `If-None-Match` header set to the value of the received `ETag`. The client also provides the `Wait` header with a timeout value in seconds.

{% highlight http %}
GET /object HTTP/1.1
If-None-Match: "b1946ac9"
Wait: 60
...
{% endhighlight %}

If the data changes while the request is open, then the new data is returned:

{% highlight http %}
HTTP/1.1 200 OK
ETag: "2492d234"
Link: </object>; rel=value-wait
...
{% endhighlight %}

If the data does not change, then a 304 is returned:

{% highlight http %}
HTTP/1.1 304 Not Modified
ETag: "b1946ac9"
Link: </object>; rel=value-wait
{% endhighlight %}

If the object is deleted while the request is open, then a 404 is returned:

{% highlight http %}
HTTP/1.1 404 Not Found
...
{% endhighlight %}

### Collections

Like objects, collection resources announce support via Link headers:

    GET /collection/ HTTP/1.1
    ...

    HTTP/1.1 200 OK
    Link: </collection/?after=1395174448&max=50>; rel="changes changes-wait"
    ...

Unlike objects which use ETags, collections encode a checkpoint in the provided "changes" and/or "changes-wait" URIs. The client should request against these URIs to receive updates to the collection. The changes-wait URI is used for long-polling.

Collection resources always return items in the collection. The changes URIs should return all items that were modified after some recent checkpoint (such as the current time). If there are items, they should be returned with code 200. If there are no such items, an empty list should be returned.

The changes URIs MAY have a limited validity period. If the server considers a URI too old to process, it can return 404, which should signal the client to start over and obtain fresh changes URIs.

For realtime updates, the client provides a Wait header to make the request act as a long poll:

    GET /collection/?after=1395174448&max=50 HTTP/1.1
    Wait: 60
    ...

Any request can be made against the collection to receive a changes URIs. For example, a news feed may want to obtain the most recent N news items and be notified of updates going forward. To accomplish this, the client could make a request to a collection resource of news items, perhaps with certain query parameters indicating an "order by created_time desc limit N" effect. The client would then receive these items and use them for initial display. The response to this request would also contain changes URIs though, which the client could then begin long-polling against to receive any changes.

This spec does not dictate the format of a collections response, but it does make the following requirements: 1) Elements in a collection MUST have an id value somewhere, that if concatenated with the collection resource would produce a direct link to the object it represents, and 2) Elements in a collection SHOULD have a deleted flag, to support checking for deletions. Clients must be able to understand the format of the collections they interact with, and be able to determine the id or deleted state of any such items.

### Webhooks

A resource can indicate support for update notifications via callback by including a "callback" Link type:

    GET /object HTTP/1.1
    ...

    HTTP/1.1 200 OK
    ETag: "b1946ac9"
    Link: </object/subscription/>; rel=value-callback
    ...

The link points to a collection resource that manages callback URI registrations. The behavior of this endpoint follows the outline at http://resthooks.org/

To subscribe http://example.org/receiver/ to a resource:

    POST /object/subscription/ HTTP/1.1
    Content-Type: application/x-www-form-urlencoded

    callback_uri=http:%2F%2Fexample.org%2Freceiver%2F

Server responds:

    HTTP/1.1 201 Created
    Location: http://example.com/object/subscription/http:%2F%2Fexample.org%2Freceiver%2F
    Content-Length: 0

The subscription's id will be the encoded URI that was subscribed. To unsubscribe, delete the subscription's resource URI:

    DELETE /object/subscription/http:%2F%2Fexample.org%2Freceiver%2F HTTP/1.1

Update notifications are delivered via HTTP POST to each subscriber URI. The Location header is set to the value of the resource that was subscribed to. For example:

    POST /receiver/ HTTP/1.1
    Location: http://example.com/object
    ...

In the case of object resources, the body of the POST request contains the entire object value. If the object was deleted, then an empty body is sent.

For collection resources, the POST request body contains the response that would normally have been sent to a request for the changes URI. The request should also contain two Link headers, with rel=changes and rel=prev-changes. The recipient can compare the currently known changes link with the prev-changes link to ensure a callback was not missed. If it was, the client can resync by performing a GET against the currently known changes link.

### Server-Sent Events

A resource can indicate support for notifications via Server-Sent Events by including a "stream" Link type:

    GET /object HTTP/1.1
    ...

    HTTP/1.1 200 OK
    ETag: "b1946ac9"
    Link: </object/stream/>; rel=value-stream
    ...

The link points to a Server-Sent Events capable endpoint that streams updates related to the resource. The client can then access the stream:

    GET /object/stream/ HTTP/1.1
    ...

    HTTP/1.1 200 OK
    Content-Type: text/event-stream
    ...

Event ids should be used to allow recovery after disconnect. For example, an object resource might use ETags for event ids. Events are not named. For value-stream URIs, each event is a JSON value of the object itself, or an empty string if the object was deleted. For changes-stream URIs, each event is a JSON object of the same format that is normally returned when retrieving elements from the collection (e.g. a JSON list).

### Multiplexing

In order to reduce the number of needed TCP connections in client applications, servers may support multiplexing many long-polling requests or Server-Sent Events connections together. This is indicated by providing Links of type "multiplex-wait" and/or "multiplex-stream". For example, suppose there are two object URIs of interest:

    GET /objectA HTTP/1.1
    ...

    HTTP/1.1 200 OK
    ETag: "b1946ac9"
    Link: </objectA>; rel=value-wait
    Link: </multi/>; rel=multiplex-wait
    ...

    GET /objectB HTTP/1.1
    ...

    HTTP/1.1 200 OK
    ETag: "d3b07384"
    Link: </objectB>; rel=value-wait
    Link: </multi/>; rel=multiplex-wait
    ...

The client can detect that both of these objects are accessible via the same multiplex endpoint "/multi/". Both resources can then be checked for updates in a single long-polling request. This is done by passing each resource URI as a query parameter named "u". Each "u" param should be immediately followed by an "inm" param (meaning If-None-Match) to specify the ETag to check against for the preceding URI. Collection resources do not use a inm param, since the checkpoint is encoded in the URI itself.

    GET /multi/?u=%2FobjectA&inm="b1946ac9"&u=%2FobjectB&inm="d3b07384" HTTP/1.1
    Wait: 60
    ...

The multiplex response uses a special format of a JSON object, where each child member is named for a URI that has response data. For example:

    HTTP/1.1 200 OK
    Content-Type: application/liveresource-multiplex

    {
      "/objectA": {
        "code": 200,
        "headers": {
          "ETag": "\"2492d234\""
        },
        "body": { "foo": "bar" }
      }
    }

If the request is a long-polling request and only one URI has response data, then response data for the others should not be included.

Multiplexing SSE is also possible:

    GET /multi/?u=%2FobjectA&u=%2FobjectB HTTP/1.1
    Accept: text/event-stream
    ...

In this case, each message is encapsulated in a JSON object, with "uri" and "body" fields. The "uri" field contains the URI that the message is for. The "body" field contains a JSON-parsed value of what would normally have been sent over a non-multiplexed SSE connection. If the value is empty, then "body" should be set to null or not included.

### WebSockets

A resource can indicate support for notifications via WebSocket by including a "multiplex-ws" Link type:

    GET /object HTTP/1.1
    ...

    HTTP/1.1 200 OK
    ETag: "b1946ac9"
    Link: </updates/>; rel=multiplex-ws
    ...

The link points to a WebSocket capable endpoint. The client can then connect to establish a bi-directional session for handling subscriptions to resources and receiving notifications about them. The client and server must negotiate the "liveresource" protocol using Sec-WebSocket-Protocol headers. The wire protocol uses JSON-formatted messages.

Once connected, the client can subscribe to a resource:

    { "id": "1", "type": "subscribe", "mode": "value", "uri": "/objectA" }

Server acks:

    { "id": "1", "type": "subscribed" }

The client can also unsubscribe:

    { "id": "2", "type": "unsubscribe", "mode": "value", "uri": "/objectA" }

Server acks:

    { "id": "2", "type": "unsubscribed" }

The 'id' field is used to match up requests and responses. The client does not have to wait for a response in order to make more requests over the socket. The server is not required to respond to requests in order. Subscriptions can have mode "value" or "changes". More than one subscription can be established over a single connection.

The server notifies the client by sending a message of type "event". For values:

    {
      "type": "event",
      "uri": "/objectA",
      "headers": {
        "ETag": "..."
      },
      "body": { ... }
    }

For changes:

    {
      "type": "event",
      "uri": "/collection/",
      "headers": {
        "Link": "</collection/?after=1395174448>; rel=changes, </collection/?after=1395174448>; rel=prev-changes",
      },
      "body": [ ... ]
    }

Notifications do not have guaranteed delivery. The client can detect for errors and recover by fetching the original URI or the changes URI.
