Paperboy is a few software components to securely and robustly implement WebSockets in your application. WebSocket is a powerful technology but has some weaknesses and still poorly understood.

| issues with WebSockets                                  | our way                                                     |
|---------------------------------------------------------|-------------------------------------------------------------|
| no authorization/authentication defined in the protocol | token-based authorization                                   |
| no enforcement of Same Origin Policy                    | configurable allowed origins, validating on new connections |
| low level message handling                              | publish/subscribe model, notion of channels                 |
| error handling can be difficult to do right             | heartbeat to detect dead connections, reconnecting clients  |

## Channels
A traditional publish/subscribe model is implemented, frontend clients can subscribe to channels, application backend can send messages to channels and the messages are delivered to the subscribed clients. On subscription the backend is called to authorize the request.

## Components
![Architecture diagram](/paperboy.svg)

### Paperboy WebSocket server
Serves WebSocket connections, forward channel subscriptions to backend and dispatches application messages to subscribed clients.

### Backend connector
A library for your backend application to include. A connector instance is needed for your backend to talk to Paperboy and to authorize channel subscription requests.

### JavaScript client
A JS client library for your frontend application to be able to subscribe Paperboy channels.

### Messaging backend
Paperboy needs an actual messaging system to deliver it's internal and application messages. Important to note that channels/topics of the underlying messaging backend are not the same thing like application level Paperboy channels. The latter is just a 'virtual' abstraction and messages on every Paperboy channel are delivered through a single channel/topic in the underlying messaging backend.
Supported backends:
* Redis
* RabbitMQ

## Token-based authorization
In Paperboy a client can only subscribe to a channel with a valid token generated for the requester. The token is obtained from the application backend
via REST, and on each channel subscription paperboy verifies it. This mechanism is abstracted away as much as possible but the developer still needs to
configure a REST service on their backend and delegate the request to the paperboy connector for token generation. Important that this REST service on the backend needs to be properly secured!

![Subscription/authorization sequence diagram](/auth-seq.svg)

### What happens when a frontend client subscribes to a Paperboy channel?
1. frontend client calls the backend via REST requesting a token for the channel
2. backend authenticates the call and generates a new valid token encapsulating the authenticated user and the requested channel as token claims
3. frontend client opens the WebSocket connection and sends the token over to the WebSocket server
4. WebSocket server publishes a Redis message of the user's channel subscription request
5. backend receives the subscription request from Redis, verifies the token and publishes a subscription authorized message on Redis
6. WebSocket server receives this authorized message, flags the subscription as authorized and starts dispatching channel messages to the client

The client re-uses the WebSocket connection, all the channel subscriptions and messages are transferred on a single connection.

## Integrating Paperboy in your stack
### WebSocket server
The WebSocket server is a nodejs application, you can deploy standalone or containerized in a kubernetes cluster.
```
git clone https://github.com/gadget/paperboy-node-server.git
cd paperboy-node-server
npm install
node paperboy.js
```

For more details see in [repo](https://github.com/gadget/paperboy-node-server)

### Application frontend
Include the JS library in your frontend code and instantiate a client.
```
var paperboyClient = new PaperboyClient(tokenUrl, wsUrl);
paperboyClient.subscribe(channel, function msgHandler(msg) {
  // TODO: process msg
});
```

For more details see in [repo](https://github.com/gadget/paperboy-client)

### Application backend
Include the connector library in your backend code and configure it.
e.g. Java connector backed with Redis in a Spring boot application
```
@Bean
public JedisPool jedisPool() {
    return new JedisPool(new JedisPoolConfig(), "localhost");
}

@Bean
public PaperboyCallbackHandler paperboyCallbackHandler() {
    return new ChatAppCallbackHandler();
}

@Bean
public PaperboyConnector paperboyConnector() {
    PaperboyConnector connector = new PaperboyConnector(new RedisBackend(jedisPool), callbackHandler);
    connector.init();
    return connector;
}
```
For Java connector see in [repo](https://github.com/gadget/paperboy-connector-java)

### Messaging backend
If you don't have any of the supported systems in your stack yet, try using Redis. You can start a local instance for development:
```
docker pull redis
docker run -p 6379:6379 -d redis
```

## Scaling

## Security checklist
* Paperboy should use WebSocket over SSL: authorization tokens are sent over the WS connection, therefore it's essential to protect that traffic from MITM attacks
* Authenticate the REST endpoint used for token generation: only logged-in users in your application should be able to generate tokens
* Let the WebSocket server know which exact origins are allowed, do not use the * wildcard

## Examples
* [Simple chat application](https://github.com/gadget/paperboy-example-chat)
