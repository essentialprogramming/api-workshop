## Server-Sent Events with Spring

A popular choice for sending real-time data from the server to a web application is [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API). WebSocket opens a bidirectional connection between client and server. Both parties can send and receive messages. In scenarios where the application only needs one-way communication from the server to the client, a simpler alternative exists Server-Sent Events (SSE). It's a [HTML5 standard](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events) and utilizes HTTP as the transport protocol, the protocol only supports text messages, and it's unidirectional, only the server can send messages to the client.

### Suport

Server-Sent Events are supported in [most modern browsers](https://caniuse.com/#search=eventsource). Only Microsoft's browser IE does not have a built-in implementation. Fortunately, this is not a problem because Server-Sent Events use standard HTTP connections and can, therefore, implemented with a polyfill. The following polyfill libraries area available and add Server-Sent Event support to IE.

- https://github.com/remy/polyfills/blob/master/EventSource.js by Remy Sharp
- https://github.com/rwaldron/jquery.eventsource by Rick Waldron
- https://github.com/amvtek/EventSource by AmvTek
- https://github.com/Yaffle/EventSource by Yaffle

With SSE, the server cannot immediately start sending messages to the client. It's always the client (browser) that establishes the connection, and after that, the server can send messages. An SSE connection is a long streaming HTTP connection.

1. Client opens the HTTP connection
2. Server sends zero, one or more messages over this connection.
3. Connection is closed by the server or due to a network error.
4. Client opens a new HTTP connection and so on...

The useful thing about Server-Sent Events is that they have a built-in reconnection feature when the client loses the connection. He tries to reconnect to the server automatically. WebSocket does not have such built-in functionality.

Even though only the server can send messages to the client over SSE, you could develop applications like a chat application with this technology. Because a client application can always open a second HTTP connection with the [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) or [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) and send data to the server.

### Client API

The Server-Sent Events API in the browser is simple and consist of only one object: EventSource
To open a connection, an application needs to instantiate the object and provide the URL of the server endpoint.

`const eventSource = new EventSource('http://localhost:8080/stream'); `

The browser immediately sends a GET request to the URL with the accept header `text/event-stream`
```
GET /stream HTTP/1.1
 Accept: text/event-stream
 ...
```
Because this is a normal GET request, an application could add query parameters to the URL like with any other GET request.

`const eventSource = new EventSource('http://localhost:8080/stream?event=news'); `

Query parameters cannot be changed during the lifecycle of the EventSource object. Every time the client reconnects he uses the same URL. But an application can always close the connection with the close() method and instantiate a new `EventSource` object.

```
let eventSource = new EventSource('http://localhost:8080/stream?event=worldnews'); 
...
eventSource.close();
eventSource = new EventSource('http://localhost:8080/stream?event=sports'); 
```

The HTTP response to a EventSource GET request must contain the Content-Type header with the value `text/event-stream` and the response must be encoded with UTF-8.

```
HTTP/1.1 200
Content-Type: text/event-stream;charset=UTF-8
```

The protocol that SSE uses is text-based, starts with a keyword, then a colon (:), and a string value.
The `data` keyword specifies a message for the client. Spaces before and after the message are being ignored. Every line is separated with a carriage return (0d) or a line feed (0a) or both (0d 0a).

`data: the server message`

The server can split a message over several lines

```
data:line1
data:line2
data:line3
```
The browser concatenates these three lines and emits one event. To separate the message from each other, the server needs to send a blank line after each message.

```
data:{"heap":148713928,"nonHeap":49888752,"ts":1488640735925}

data:{"heap":149344096,"nonHeap":49889392,"ts":1488640736927}

data:{"heap":149344096,"nonHeap":49889392,"ts":1488640736929}
```

You see that the payload does not have to be a simple string. It's perfectly legal to send JSON strings and parse them on the client with `JSON.parse`.

To process these events in the browser, an application needs to register a listener for the `message` event. The property `data` of the event object contains the message. The browser filters out the keyword `data` and the colon and only assigns the string after the colon to `event.data`.

```
eventSource.onmessage = event => {
  const msg = JSON.parse(event.data);
  //access msg.ts, msg.heap, msg.nonHeap
};
```

An application can listen for the `open` and `error` event. The `open` event is emitted as soon as the server sends a 200 response back. The `error` event is fired whenever a network error occurs. It is also emitted when the server closes the connection.

```
eventSource.onopen = e => console.log('open');
eventSource.onerror = e => {
  if (e.readyState == EventSource.CLOSED) {
   console.log('close');
  }
  else {
    console.log(e);
  }
};
```