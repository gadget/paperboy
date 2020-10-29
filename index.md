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
TODO: bird's eye view on components

### Channels
TODO:
publisher is the backend
subscriber is the frontend

### Token-based authorization
In Paperboy a client can only subscribe to a channel with a valid token generated for the requester. The token is obtained from the application backend
via REST, and on each channel subscription paperboy verifies it. This mechanism is abstracted away as much as possible but the developer still needs to
configure a REST service on their backend and delegate the request to the paperboy connector for token generation. Important that this REST service on the backend needs to be properly secured, only authorized users should be able to generate tokens!
![Subscription/authorization sequence diagram](/auth-seq.png)

### Integrating Paperboy in your stack
The WebSocket server is a nodejs application, you can deploy standalone or containerized in a kubernetes cluster.
TODO: examples
TODO: frontend
Your backend application needs to include a paperboy-connector library to make it able to talk to Paperboy.
TODO: example with Java
