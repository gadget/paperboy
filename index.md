## Welcome to Paperboy
Paperboy is a few software components to securely and robustly implement WebSockets in your application. WebSocket is a powerful technology but has some weaknesses and still poorly understood. The goal is to hide the complexity of authorization from the developer and leave no room for mistake.

| issues with WebSockets                                  | our way                                                     |
|---------------------------------------------------------|-------------------------------------------------------------|
| no authorization/authentication defined in the protocol | token-based authorization                                   |
| no enforcement of Same Origin Policy                    | configurable allowed origins, validating on new connections |
| low level message handling                              | publish/subscribe model, notion of channels                 |
| error handling can be difficult to do right             | heartbeat to detect dead connections, reconnecting clients  |
| TODO: message validation                                |                                                             |

It can also be a good fit in case you want to offload WebSocket traffic from your main backend and scale it independently.

### Architecture
**Paperboy WebSocket server**
Serves WebSocket connections, forward channel subscriptions to backend and dispatch application messages to connected peers.
**Backend connector library**
Your backend needs to include the connector library to be able to talk to Paperboy and to authorize channel subscription requests.
**JavaScript client library**
Should be part of your frontend codebase. See below how to use.

**Redis backend**

### Channels
TODO:

### Token-based authorization
In Paperboy a client can only subscribe to a channel with a valid token generated for the requester. The token is obtained from the application backend
via REST, and on each channel subscription paperboy verifies it. This mechanism is abstracted away as much as possible but the developer still needs to
configure a REST service on their backend and delegate the request to the paperboy connector for token generation. Important that this REST service on the backend needs to be properly secured!

![Subscription/authorization sequence diagram](/auth-seq.png)

### Integrating Paperboy in your stack
**WebSocket server**
The WebSocket server is a nodejs application, you can deploy standalone or containerized in a kubernetes cluster.

TODO: examples

**Application frontend**
Just copy paperboy-client.js under your frontend project, include it in your pages and start using it like below.

```
paperboyClient = new PaperboyClient(tokenBaseUrl, wsUrl, channel, function msgHandler(msg) {
  // TODO: process msg
});
paperboyClient.subscribe();
```

**Application backend**
Include a connector library in your backend.
```
e.g. Java connector as a gradle dependency
implementation 'com.paperboy.connector:paperboy-connector-java:1.0-SNAPSHOT'
```

Setup a connector instance, implement your callbackHandler for authorization.
```
e.g. Spring bean of the connector instance

@Bean
public PaperboyConnector paperboyConnector() {
    PaperboyConnector connector = new PaperboyConnector(jedisPool, callbackHandler);
    connector.startListening();
    return connector;
}
```

Expose token generation as a REST endpoint. Path should be secured!
```
e.g. Spring REST endpoint for token generation

@GetMapping(path = "/paperboyAuth/{channel}")
public String requestToken(Principal principal, @PathVariable String channel) {
  return paperboyConnector.requestToken(principal, channel);
}
```

### Security checklist
* Paperboy should use WebSocket over SSL: authorization tokens are sent over the WS connection, therefore it's essential to protect that traffic from MITM attacks
* Authenticate the REST endpoint used for token generation: only logged-in users in your application should be able to generate tokens
* Let the WebSocket server know which exact origins are allowed, do not use the * wildcard
