# LiveResource Protocol

LiveResource is a protocol for receiving live updates of web resources. Resources may be of any kind, for example REST API endpoints or files. The protocol aims to to be simple, minimal, and align with HTTP/REST conventions.

## Design goals

* Given a resource URL, it should be possible for a receiving entity to listen for updates of the resource at that URL.
* It should be possible for the server to deliver updates instantly, without the recipient needing to poll.
* It should be possible to listen for three types of updates:
    1. The entire content of a resource.
    2. Only the changed content of a resource.
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

```http
GET /resource HTTP/1.1
Host: example.org
```

The response from the server may include `LiveResource-Property` headers and/or `Link` headers indicating various live update mechanisms:

```http
HTTP/1.1 200 OK
ETag: "b1946ac9"
LiveResource-Property: wait; min=20
Link: </resource>; rel=alternate; type=text/event-stream
Link: </resource/subscriptions/>; rel=http://liveresource.org/protocol/callbacks
...
```

This information can also be obtained by making a HEAD request, if the client wants to see what a resource supports without having to retrieving its content.

The client can then listen for updates by making use of the advertised mechanisms. In the above example, the supplied headers would mean:

* Long-polling for updates is possible by making a GET request to `http://example.org/resource/`, with request headers `If-None-Match: "b1946ac9"` and `Prefer: wait={timeout}`.
* Receiving a Server-Sent Events stream is possible by making a GET request to `http://example.org/resource/`, with request header `Accept: text/event-stream`.
* Webhooks may be registered by making a POST request to `http://example.org/resource/subscriptions/`, with parameter `callback_uri` set to the URL to receive updates.

## Update types

There are three update types:

* Value: The entire value of a resource, headers and content.
* Changes: Data indicating the changed parts of a resource. The LiveResource protocol does not define a specific format for changes. The server may use any data format to represent changes of a resource. For example, it could return a subset of items in the case of an updated collection, or a text diff, or a JSON patch to show the changes of a JSON object, etc. It is up to clients to know how to interpret such payloads.
* Hint: An indication that the resource has changed without providing any of resource data. When a client receives a hint, it will need to make a separate GET request to retrieve the data.

When a client listens for updates, it chooses to listen for either value updates or changes updates. The use of hints is decided by the server. For example, a client might listen for value updates, but the server could choose to deliver hints rather than values, or potentially a mix of both hints and values.

## Update mechanisms

There are five update mechanisms:

* Plain-polling: The client polls the resource on an interval to retrieve the latest data.
* Long-polling: The client makes a request and the server waits until the resource has changed before responding.
* Stream: The client listens for updates using the Server-Sent Events protocol.
* Socket: The client listens for updates using WebSockets.
* Callback: The receiver registers a callback URL that the server should make requests to when the resource has changed.

## Resource properties

`LiveResource-Property` headers are used to indicate capabilities of the resource itself.

Property         | Meaning
---------------- | ----------------------------------
`wait`           | The resource supports long-polling.
`multiplex=type` | The resource URI may be used with a multiplex transport. Allowed values are `request`, `socket`, `callbacks`, and `required`. More than one value can be provided using a quoted concatenated string, e.g. `multiplex="request socket"`. The `required` value is not a multiplex transport type, but a special value that indicates the resource MUST be used with a multiplex transport.

Note: only the long-polling mechanism is advertised as a property of the resource itself. The stream, socket, and callback mechanisms usually involve working with different URIs, and so `Link` headers are used for those instead (see next section).

## Links

`Link` headers are used to indicate capabilities available at potentially different URIs than the resource of interest.

Link rel           | Meaning
------------------ | ----------------------------------------------------------------------------------------------------
`http://liveresource.org/protocol/changes`             | Changes to the resource. The URI will almost always include some kind of checkpoint information, e.g. `/resource/?after=1`.
`alternate`                                            | This can be used to specify an alternate location to use for long-polling (by including a `wait-min` parameter as indication), or the location to use for the stream mechanism (by including a `type` parameter set to `text/event-stream`).
`http://liveresource.org/protocol/callbacks`           | Base URI for subscription management of HTTP callback (webhook) updates of the entire value of the resource.
`http://liveresource.org/protocol/changes-callbacks`   | Base URI for subscription management of HTTP callback (webhook) updates of the changed parts of the resource.
`http://liveresource.org/protocol/multiplex-request`   | Base URI for accessing multiple resources in a single request. This URI MUST support long-polling.
`http://liveresource.org/protocol/multiplex-socket`    | Base URI for accessing multiple resources via a single WebSocket connection.
`http://liveresource.org/protocol/multiplex-callbacks` | Base URI for subscription management of HTTP callback (webhook) updates of multiple resources.

The custom relationship types have very long names because they are not registered with the IANA yet. The intent is that types could eventually be registered using the leaf part (e.g. `changes`, `callbacks`, `multiplex-socket`, etc).

## Protocol

### Plain-polling

For plain polling, the client repeatedly retrieves the resource URI on an interval. By default, it is RECOMMENDED that clients don't poll a URL more frequently than once every 2 minutes. Servers MAY indicate a preferred interval by providing an `X-Poll-Interval` response header with a value in seconds. Clients SHOULD allow configuring an interval to use in situations where the server does not provide `X-Poll-Interval` *and* the client knows that a non-default interval is acceptable.

For value updates, the client polls the resource URI.

For changes updates, the client polls a `http://liveresource.org/protocol/changes` link if provided.

This protocol does not specify a way to receive hint updates with plain-polling.

### Long-polling

Resources may announce support for value updates via long-polling by including `ETag` and `LiveResource-Property` headers:

```http
GET /object HTTP/1.1
...

HTTP/1.1 200 OK
ETag: "b1946ac9"
LiveResource-Property: wait; min=20
...
```

To make use of long-polling the client issues a GET to the resource URI with the `If-None-Match` header set to the value of the received `ETag`. The client also provides the `Prefer` header with a `wait` timeout value in seconds.

```http
GET /object HTTP/1.1
If-None-Match: "b1946ac9"
Prefer: wait=60
...
```

If the data changes while the request is open, then the new data is returned:

```http
HTTP/1.1 200 OK
ETag: "2492d234"
LiveResource-Property: wait; min=20
...
```

If the data does not change, then 304 is returned:

```http
HTTP/1.1 304 Not Modified
ETag: "b1946ac9"
```

If the object is deleted while the request is open, then a 404 is returned:

```http
HTTP/1.1 404 Not Found
...
```

Changes are supported in a similar way. The resource announces support for changes via a `Link` header:

```http
HEAD /collection/ HTTP/1.1
...

HTTP/1.1 200 OK
Link: </collection/?after=1395174448&max=50>; rel=http://liveresource.org/protocol/changes
...
```

Before the changes link can be long-polled, the client needs to know whether or not it supports it. This can be done two ways. One is to request the changes link and look for the `wait` property:

```http
HEAD /collection/?after=1395174448&max=50 HTTP/1.1
...

HTTP/1.1 200 OK
LiveResource-Property: wait; min=20
...
```

The other way is for the server to hint at this capability in advance, using the `wait-min` link parameter:

```http
HEAD /collection/ HTTP/1.1
...

HTTP/1.1 200 OK
Link: </collection/?after=1395174448&max=50>; rel=http://liveresource.org/protocol/changes; wait-min=20
...
```

This way a separate discovery request is avoided.

Unlike value updates which use ETags, changes updates encode checkpoint information in the provided changes URI. The client should request against this URI to receive updates of the resource.

The changes URI should return all changes that have occurred after some recent checkpoint (such as a timestamp). If no changes have occurred, the response must have a way to indicate this (e.g. empty body, empty JSON object or list, etc). Regardless of whether there were changes or not, the response code SHOULD be 200.

The changes URIs MAY have a limited validity period. If the server considers a URI too old to process, it can return 404, which should signal the client to start over and obtain a fresh changes URI.

The client provides a `Prefer` header to make the request act as a long poll:

```http
GET /collection/?after=1395174448&max=50 HTTP/1.1
Prefer: wait=60
...
```

Any request can be made against a resource to receive a changes URI. For example, a news feed may want to obtain the most recent N news items and be notified of updates going forward. To accomplish this, the client could make a request to a collection resource of news items, perhaps with certain query parameters indicating an "order by created time desc limit N" effect. The client would then receive these items and use them for initial display. The response to this request would also contain a changes URI, which the client could then begin long-polling against to receive any changes.

### Stream

A resource can indicate support for notifications via Server-Sent Events by including an `alternate` link for content type `text/event-stream`:

```http
GET /object HTTP/1.1
...

HTTP/1.1 200 OK
ETag: "b1946ac9"
Link: </object/stream/>; rel=alternate; type=text/event-stream
...
```

The link points to a Server-Sent Events capable endpoint that streams updates related to the resource. The client can then access the stream:

```http
GET /object/stream/ HTTP/1.1
Accept: text/event-stream
...

HTTP/1.1 200 OK
Content-Type: text/event-stream
...
```

Event IDs MAY be used to allow recovery after disconnect. For example, a stream of value updates might use ETags for event IDs. Events have name `update`. The first line of the event data are the headers encoded in a JSON object. The second line and any lines after that is the resource content. The content SHOULD be the same as would be returned with regular GET requests, for value or changes.

Streamed hint updates simply have no data.

### Callbacks

A resource can indicate support for update notifications via callbacks (webhooks) by including link relationships of `http://liveresource.org/protocol/callbacks` and/or `http://liveresource.org/protocol/changes-callbacks`.

```http
GET /object HTTP/1.1
...

HTTP/1.1 200 OK
ETag: "b1946ac9"
Link: </object/subscriptions/>; rel=http://liveresource.org/protocol/callbacks
...
```

The link points to a collection resource that manages callback URI registrations. The behavior of this endpoint follows the outline at [http://resthooks.org/](http://resthooks.org).

To subscribe `http://example.org/receiver/` to a resource:

```http
POST /object/subscriptions/ HTTP/1.1
Content-Type: application/x-www-form-urlencoded

callback_uri=http:%2F%2Fexample.org%2Freceiver%2F
```

Server responds:

```http
HTTP/1.1 201 Created
Location: http://example.com/object/subscriptions/http:%2F%2Fexample.org%2Freceiver%2F
Content-Length: 0
```

The subscription's id will be the encoded URI that was subscribed. To unsubscribe, delete the subscription's resource URI:

```http
DELETE /object/subscriptions/http:%2F%2Fexample.org%2Freceiver%2F HTTP/1.1
```

Update notifications are delivered via HTTP POST to each subscriber URI. The `Location` header is set to the value of the resource that was subscribed to. For example:

```http
POST /receiver/ HTTP/1.1
Host: example.org
Location: http://example.com/object
...
```

In the case of value updates, the body of the POST request contains the entire value. If the resource was deleted, then an empty body is sent.

For changes updates, the POST request body contains the response that would normally have been sent to a request for the changes URI. The request should also contain two additional headers, `Changes-Id` and `Previous-Changes-Id`. The recipient can compare the currently known changes ID with the previous changes ID to ensure a callback was not missed. If it was, the client can resync by performing a GET against the currently known `http://liveresource.org/protocol/changes` link.

### Multiplexing

In order to reduce the number of needed TCP connections in client applications (or the number of callback registration endpoints in the case of callbacks), servers may support multiplexing connections or callback registration endpoints to handle multiple URIs at the same time. This is indicated by providing links relationships `multiplex-request`, `multiplex-socket`, and/or `multiplex-callbacks`. For example, suppose there are two resource URIs of interest:

```http
GET /objectA HTTP/1.1
...

HTTP/1.1 200 OK
ETag: "b1946ac9"
LiveResource-Property: wait; min=20, multiplex=request
Link: </multi/>; rel=http://liveresource.org/protocol/multiplex-request; wait-min=20
...

GET /objectB HTTP/1.1
...

HTTP/1.1 200 OK
ETag: "d3b07384"
LiveResource-Property: wait; min=20, multiplex=request
Link: </multi/>; rel=http://liveresource.org/protocol/multiplex-request; wait-min=20
...
```

The client can detect that both of these resources are accessible via the same multiplex-request endpoint `/multi/`. Both resources can then be checked for updates in a single long-polling request. This is done by passing each resource URI as in a `Uri` request header. For values updates, each `Uri` header should contain an `If-None-Match` parameter specifying the ETag to check against. For changes updates, the `If-None-Match` parameter is not needed, since the checkpoint information is encoded in the URI itself.

```http
GET /multi/ HTTP/1.1
Uri: </objectA>; If-None-Match="b1946ac9"
Uri: </objectB>; If-None-Match="d3b07384"
Prefer: wait=60
...
```

The multiplex response uses a special format of a JSON object, where each child member is named for a URI that has response data. For example:

```http
HTTP/1.1 200 OK
Content-Type: application/liveresource-multiplex

{
  "/objectA": {
    "code": 200,
    "headers": {
      "ETag": "\"2492d234\""
    },
    "body": "{ \"foo\": \"bar\" }"
  }
}
```

If the request is a long-polling request and only one URI has response data, then response data for the others SHOULD NOT be included.

For WebSockets, a resource can indicate support for `socket` multiplexing:

```http
GET /object HTTP/1.1
...

HTTP/1.1 200 OK
ETag: "b1946ac9"
LiveResource-Property: multiplex=socket
Link: </ws/>; rel=http://liveresource.org/protocol/multiplex-socket
...
```

The link points to a WebSocket capable endpoint. The client can then connect to establish a bi-directional session for handling subscriptions to resources and receiving notifications about them. The client and server MUST negotiate the `liveresource` protocol using `Sec-WebSocket-Protocol` headers. The wire protocol uses JSON-formatted messages.

Once connected, the client can subscribe to a resource:

```
GET /objectA
```

Server acks:

```
200 /objectA
```

The client can also unsubscribe:

```
CANCEL /objectA
```

Server acks:

```
200 /objectA
```

The client does not have to wait for a response in order to make more requests over the socket. The server is not required to respond to requests in order. More than one subscription can be established over a single connection.

The server notifies the client by sending a message:

```
* /objectA {"ETag": "..."}
{ ... }
```

The first line includes any updated headers of the resource encoded in JSON. All remaining lines comprise the resource content.

Hint updates contain only the first line and without the headers part.

For sharing a callback registration endpoint with many resources, a resource can indicate support for `callbacks` multiplexing:

```http
GET /object HTTP/1.1
...

HTTP/1.1 200 OK
ETag: "b1946ac9"
LiveResource-Property: multiplex=callbacks
Link: </hooks/>; rel=http://liveresource.org/protocol/multiplex-callbacks
...
```

When the client registers a callback URI by making a POST request to the `/hooks/` endpoint, it includes a `uri` parameter specifying the resource of interest.
