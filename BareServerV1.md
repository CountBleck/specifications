# Bare Server V1 Endpoints

All endpoints are prefixed with `/v{version}/`

The endpoint `/` on V1 would be `/v1/`

## Request the server to fetch a URL from the remote.

| Method | Endpoint   |
| ------ | ---------- |
| `*`    | /       |

Request Body:

The body will be ignored if the request was made as `GET`. The request body will be forwarded to the remote request made by the bare server.

Request Headers:

```
X-Bare-Host: example.org
X-Bare-Port: 80
X-Bare-Protocol: http:
X-Bare-Path: /index.php
X-Bare-Headers: {"Accept":"text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9"}
X-Bare-Forward-Headers: ["accept-encoding","accept-language"]
```

The headers below are required. Not specifying a header will result in a 400 status code. All headers are not tampered with, whatever is specified will go directly to the destination.

- X-Bare-Host: The host of the destination WITHOUT the port. This would be equivalent to `URL.hostname` in JavaScript.
- X-Bare-Port: The port of the destination. This must be a valid number. This is not `URL.port`, rather the client needs to determine what port a URL goes to. An example of logic done a the client: the protocol `http:` will go to port 80 if no port is specified in the URL.
- X-Bare-Protocol: The protocol of the destination. V1 bare servers support `http:` and `https:`. If the protocol is not either, this will result in a 400 status code.
- X-Bare-Path: The path of the destination. Be careful when specifying a path without `/` at the start. This may result in an error from the remote.
- X-Bare-Headers: A JSON-serialized object containing request headers. Request header names may be capitalized. When making the request to the remote, capitalization is kept. Consider the header capitalization on `HTTP/1.0` and `HTTP/1.1`. Sites such as Discord check for header capitalization to make sure the client is a web browser.
- X-Bare-Forward-Headers: A JSON-serialized array containing names of case-insensitive request headers to forward to the remote. For example, if the client's useragent automatically specified the `Accept` header and the client can't retrieve this header, it will set X-Bare-Forwarded-Headers to `["accept"]`. The Bare Server will read the `accept` header from the request headers (not X-Bare-Headers`) and add it to the headers sent to the remote. The server will automatically forward the following headers: Content-Encoding, Content-Length, Transfer-Encoding

The headers below are optional and will default to `[]` if they are not specified. They are intended to be used for caching purposes.

- X-Bare-Pass-Headers: A JSON-serialized array of case-insensitive headers. If these headers are present in the remote response, the values will be added to the server response.
The list must not include the following: `vary`, `connection`, `transfer-encoding`, `access-control-allow-headers`, `access-control-allow-methods`, `access-control-expose-headers`, `access-control-max-age`, `access-control-request-headers`, `access-control-request-method`.
- X-Bare-Pass-Status: A JSON-serialized array of HTTP status codes, like 204 and 304. If the remote response status code is present in this list, the server response status will be set to the remote response status.

Response Headers:

```
Content-Encoding: ...
Content-Length: ...
X-Bare-Status: 200
X-Bare-Status-text: OK
X-Bare-Headers: {"Content-Type": "text/html"}
```

- Content-Encoding: The remote body's content encoding.
- Content-Encoding: The remote body's content length.
- X-Bare-Status: The status code of the remote.
- X-Bare-Status-Text: The status text of the remote.
- X-Bare-Headers: A JSON-serialized object containing remote response headers. Response headers may be capitalized if the remote sent any capitalized headers.

Response Body:

The remote's response body will be sent as the response body.

A random character sequence used to identify the WebSocket and it's metadata. 

## Request a new WebSocket ID.

| Method | Endpoint     |
| ------ | ------------ |
| `GET`  | /ws-new-meta |

Response Headers:

```
Content-Type: text/plain
```

Response Body:

The response is a unique sequence of hex encoded bytes.

```
ABDCFE009023
```

## Request the server to create a WebSocket tunnel to the remote.

| Method | Endpoint  |
| ------ | --------- |
| `GET`  | /         |

Request Headers:

```
Upgrade: websocket
Sec-WebSocket-Protocol: bare, ...
```

- Sec-WebSocket-Protocol: This header contains 2 values: a dummy protocol named `bare` and an encoded, serialized, JSON object.

The JSON object looks like:

```json
{
	"remote": {
		"host": "example.org",
		"port": 80,
		"path": "/ws-path",
		"protocol": "ws:"
	},
	"headers": {
		"Origin": "http://example.org",
		"Sec-WebSocket-Protocol": "original_websocket_protocol"
	},
	"forward_headers": [
		"accept-encoding",
		"accept-language",
		"sec-websocket-extensions",
		"sec-websocket-key",
		"sec-websocket-version"
	],
	"id": "UniqueID_123"
}
```

This serialized JSON is then encoded. See [WebSocketProtocol.md](https://github.com/tomphttp/specifications/blob/master/WebSocketProtocol.md) for in-depth on this encoding.

- remote {Object}: An object similar to `X-Bare-` headers used when making a request.
- headers {Object}: An object similar to `X-Bare-headers`
- forward_headers {Array}: An array similar to `X-Bare-Forward-Headers`. These are all lowercase but may reference capitalized headers.
- id {String}?: The unique id generated by the server. If this field isn't specified, no meta data will be stored, making the response headers inaccessible to the client.

Response Headers:

```
Sec-WebSocket-Accept: bare
```

Sec-WebSocket-Accept: The remote's accept header.

Sec-WebSocket-Protocol: The first value in the list of protocols the client sent.

All headers that aren't listed above are irrelevant. The WebSocket class in browsers can't access response headers.

Response Body:

The response is a stream, forwading bytes from the remote to the client. Once either the remote or client close, the remote and client will close.

## Request the metadata for a specific WebSocket

| Method | Endpoint |
| ------ | -------- |
| `GET`  | /ws-meta |

Request Headers:

```
X-Bare-ID: UniqueID_123
```

- X-Bare-ID: The unique ID returned by the server in the pre-request.

> ⚠ All WebSocket metadata is cleared after requesting the metadata or 30 seconds after the connection was established.

An expired or invalid X-Bare-ID will result in a 400 status code.

Response Headers:

```
Content-Type: application/json
```

Response Body:

```json
{
	"headers": {
		"Set-Cookie": [
			"Cookie",
			"Cookie"
		],
		"Sec-WebSocket-Accept": "original_websocket_protocol"
	}
}
```
