# Gateway Specification

## Overview

```
 Internet────────────────────────────────────────────┐      Internal infrastructure (e.g. AWS)────────────────────┐
 │                                                   │      │                                                     │
 │                                                   │      │                                                     │
 │                                                   │      │                                                     │
 │                                                   │      │            ┌───┐         ┌───┐                      │
 │                                                   │      │            │ λ │         │ λ │                      │
 │                                                   │      │            └───┘         └───┘                      │
 │                                                   │      │              │             ▲                        │
 │         ┌─────────────────┐                       │      │           publish      react on                     │
 │         │                 │                       │      │           events        events                      │
 │         │   Mobile apps   │◀───HTTP & push    ┌───┴──────┴────────┐ (pub/sub)    (pub/sub)                     │
 │         │                 │   notifications   │┌───────┐          │     │             │                        │
 │         └─────────────────┘         │         ││       │          │     │             │                        │
 │                                     │         ││       │          │     │             │                        │
 │                                     │         ││ Edge  │   Gateway│     │             │    function   ┌───┐    │
 │         ┌─────────────────┐         ├────────▶││ proxy │          │─────┴─────────┬───┴────metadata ─▶│ λ │    │
 │         │                 │         │         ││       │          │               │       (discovery) └───┘    │
 │         │  Browser apps   │◀───GraphQL &      ││       │          │               │                     │      │
 │         │                 │    WebSockets     │└───────┘          │               │                     │      │
 │         └─────────────────┘                   └───┬──────┬────────┘           configure              call a    │
 │                                                   │      │                    function              function   │
 │                                                   │      │                 (config store)             (FDK)    │
 │                                                   │      │                        │                     │      │
 │                                                   │      │                        ▼                     ▼      │
 │                                                   │      │                      ┌───┐                 ┌───┐    │
 │                                                   │      │                      │ λ │                 │ λ │    │
 │                                                   │      │                      └───┘                 └───┘    │
 │                                                   │      │                                                     │
 │                                                   │      │                                                     │
 └───────────────────────────────────────────────────┘      └─────────────────────────────────────────────────────┘
```

## Motivation

- enable developers to build FaaS backends for modern web applications by providing event-based communication layer
- enable developers to build micro-services (serverless) architectures by providing discovery features and communication layer
- enable developers to build event-driven backend systems by providing configuration store and communication layer

## Concepts

### Gateway

Gateway is a single product with a unique set of features that enable building serverless event-driven systems. Features:

- event discovery - registry of events that occur in the system
- pub/sub - lightweight pub/sub system allowing reacting (with functions) on events registered in event discovery
- function discovery - register of functions that can be discovered by other functions or legacy systems
- edge proxy - exposing public GraphQL/HTTP/REST/WebSockets endpoints that allow communicating with backend functions and events
- config store - key/value store for dynamic configuration and feature flags
- ACL tokens - access control list system used to control access to functions and other resources

What Gateway is NOT:

- it's not a replacement for message queues (no message ordering, strong durability guarantees, load-balanced message delivery)
- it's not a replacement for streaming platforms (no processing capabilities, consumers groups)
- it's not a replacement for existing service discovery solutions

### Event discovery

Event discovery is a sub service for storing metadata of events that are handled by the Gateway. Event discovery is a single source of truth about types of events that occur in the system. Events can be grouped into streams by prefixing name with stream name (separated with "/") e.g. "users/userCreated", "cart/itemAdded".

#### Schema

Event schema can be defined explicitly using Gateway API or implicitly with the first occurrence of the event. Schema created implicitly can be changed. After schema creation every event with the same name has to be valid against schema.

#### Access types

There are two types of access:
- private (default) - event occurs in the backend system, internally, only functions from Function discovery can subscribe to them.
- public - events that can be published to external subscribers (browser application via WebSockets, pushed to a mobile app via SNS, or pushed to external HTTP endpoint)

### Pub/Sub

Competitors: Firebase Cloud Messaging, RabbitMQ, AWS SNS

Pub/Sub sub system allows publishing and subscribing to custom events. Pub/Sub sub system is lightweight message broker tailored for FaaS architectures.

#### Publication

An event can be published via Gateway API. If there is no schema registered in Events discovery for this type of event, new schema is created ad-hoc. If the event is not valid against existing schema error is returned and the event is not published. Events can also be pulled from other systems (like Kafka, RabbitMQ) via connectors that forward events to the Gateway.

#### Subscription

A subscriber can subscribe to a specific event or group of events. Subscriptions are created using Gateway API by providing event/stream name and subscription target. Target types:

- function - target function has to be registered in Function discovery before creating the subscription (for private & public events).
- HTTP endpoint - any HTTP endpoint (for public events)
- WebSockets channel (for public events, more about WebSockets channels in Edge proxy section)
- any external source via connectors (for private & public events)

##### Connectors

A connector is a Gateway plugin that imports/exports events from/to external sources like Kafka, Kinesis or RabbitMQ. Events are either pulled or pushed by a connector. A connector has a full access (including private events) to events occurring in the system. It may be helpful for importing events from existing legacy system or for archiving/event sourcing purpose.

```
                        ┌─────────────────────┐                  ┌─────┐
                   ┌────┴────┐                │     event: ─────▶│  λ  │
┌────────────┐     │  Kafka  │                │   userCreated    └─────┘
│   Kafka    │◀───▶│connector│                │        │                
└────────────┘     └────┬────┘  Gateway       │        │                
                   ┌────┴────┐                │────────┤                
┌────────────┐     │RabbitMQ │                │        │         ┌─────┐
│  RabbitMQ  │◀───▶│connector│                │     event: ─────▶│  λ  │
└────────────┘     └────┬────┘                │   userVisited    └─────┘
                        └─────────────────────┘                                 
```

A connector can be either import, export or both. Example of archiving all events to S3 with S3 connector configured as a export connector:

```
                      ┌─────────────────────────────────┐                   
                      │             Gateway             │                   
                      │                                 │                   
                      │             ┌───┐               │                   
                      │      ┌─────▶│ λ │               │                   
┌─────────────────────┴┐     │      └───┘              ┌┴──────────────────┐
│  Kinesis connector   │     │                         │   S3 connector    │
│(configured to import │     │                         │  (configured to   │
│ events from specific │─────┴────────────────────────▶│ export all events │
│   Kinesis stream)    │                               │   to S3 bucket)   │
└─────────────────────┬┘                               └┬──────────────────┘
                      │                                 │                   
                      └─────────────────────────────────┘                   
```

#### Pub/Sub & Serverless framework

In case of Serverless Framework function can subscribe using event sources:

```
gateway:
  token: xxx
  url: gateway.acme.com

functions:
  hello:
    handler: index.run
    events:
      - gateway:
          event: userCreated
      - gateway:
          event: userDeleted
```


### Function discovery

Competitors: Consul, etcd

Function discovery is a low latency, highly available sub service storing metadata and health info about registered function. Function discovery maps functions name to function metadata (provider, provider specific ID, region, timeout, etc.). Function discovery is a source of data for FDK enabling calling functions from other functions or legacy systems.

Functions can be grouped into services/apps by prefixing name with service/app name (separated with "/") e.g. "users/create", "cart/addItem".

#### Problems to solve

- calling a function without knowing where the function is deployed (provider, region)
- calling a function that exposes HTTP endpoint (e.g. HTTP endpoint via APIG or now.sh/stdlib/clay functions) without knowing exact URL
- insight into which function are working properly, which are failing (because of timeout, throttle, or runtime error).

#### Registration

Function can be registered via Gateway API. Function types:

- FaaS function:
  - name
  - instances:
    - provider
    - provider specific ID
    - region
- function exposing HTTP
  - name
  - HTTP method
  - instances:
    - region
    - url

#### Discovery

A function can be discovered with a function name. Function discovery returns function metadata that allows calling the function.

#### Health status

Function discovery stores information about functions health. Health information is advisory and doesn't affect function metadata returned from Discovery. The function is healthy if all calls are successfully handled. The unhealthy function is a function that couldn't handle request because of timeout, throttle or runtime error.

Gateway exposes API to provide info about function health (failed calls).

#### Access types

There are two types of access:
- private (default) - functions that can be called from within the same infrastructure/system
- public - functions that can be called via edge proxy endpoints. More about public functions in "Edge proxy" section.

#### Multi-regional deployments

It's NOT mandatory to provide region during registration. It's additional data that distinguish different instances of the same function enabling users to provide low latency, reliable experience for end users.

#### FDK & function discovery

FDK uses Gateway client internally. Flow:

- `fdk.request()`, `fdk.call()` or `fdk.trigger()` call
- FDK fetches function metadata from the Gateway via Gateway client
- If function is deployed to multiple regions return metadata of the closest function instance
- FDK using ARN/URL calls the function using provider SDK (e.g. `aws-sdk`) or makes HTTP requests
- notify Gateway service about any issues during the remote call (e.g. function not found, timeout, throttled, HTTP endpoint returns an error)

### Config store

Competitors: Consul, etcd

Config store is a low latency, highly available, simple DB for storing key (string) -> value (string) pairs. It can be used to store any kind of data. The intention is to replace environment variables as a way to pass configuration values.

Values can be grouped into folders by prefixing key with folder name e.g. `users/apikey`.

#### Problems to solve

- storing configuration that can be dynamically changed without requiring function redeploy
- providing a secure way for storing credentials used by functions

#### FDK & config store

FDK provides simple say for fetching and storing data in config store.

```
fdk.set('users/twilioKey', 'xxx')
fdk.get('users/twilioKey')
```

### ACL system

*ACL system is highly inspired by [Consul's ACL system](https://www.consul.io/docs/guides/acl.html) and AWS IAM.*

ACL system can be used to control access to functions and events. The ACL is based on tokens. The ACL is capability-based, relying on tokens to which fine-grained rules can be applied.

#### Tokens

Every token has an ID, description and rule set. Tokens are bound to a set of rules that control which Gateway resources/APIs the token has access to.

#### Rules

A rule describes the policy that must be enforced. Rules are prefix-based, allowing operators to define different namespaces (config store folder, service, stream). A rule can be enforced on following Gateway APIs:

- config - config store operations
- function - function discovery operations
- event - event discovery operations
- acl - ACL system operations
- endpoint - edge proxy endpoints operations

Rules can make use of one or more policies. Policies can have following dispositions:

- read - allow the resource to be read but not modified
- write - allow the resource to be read and modified
- deny - do not allow the resource to be read or modified

With prefix-based rules, the most specific prefix match determines the action. This allows for flexible rules like an empty prefix to allow read-only access to all resources, along with some specific prefixes that allow write access or that are denied all access.

**Example**

Following rule set gives permissions for writing values in config store under `users/` folder and allows function react on `userCreated` event:

```json
{
  "id": "94f7efe8-db7d-4123-8d84-b9c75eaa495d",
  "description": "users service functions token",
  "rules": {
    "config": [{
      "resource": "users/",
      "policy": "write"
    }],
    "event": [{
      "resource": "userCreated",
      "policy": "read"
    }]
  }
}
```

### Edge proxy

Competitors: AWS API Gateway

Edge proxy (EP) exposes endpoints that can be accessed publicly (on the Internet) or internally. Endpoints types:

- HTTP endpoint - simple HTTP endpoint that allows calling functions registered in Function discovery. Similar to AWS API Gateway.
- GraphQL endpoint - this endpoint allows exposing multiple functions via GraphQL endpoint without a need to create GraphQL server. EP acts as a GraphQL server and takes care of calling backend functions. Developer is only responsible for providing GraphQL schema.
- WebSockets channels - a bridge between Pub/Sub and web browser (or any WebSockets-compatible client)
- SNS/Firebase Cloud Messaging/etc. - a bridge between Pub/Sub and mobile devices or other supported targets

Gateway exposes API for creating/deleting endpoints.

#### HTTP & GraphQL endpoints

Those endpoints accept HTTP request and forward them to backend functions (prior registering them in Function discovery).

#### WebSockets channels

WebSockets channels endpoints enable two-way communication between backend function and the browser app. Browser app can subscribe to public events defined in Event discovery. It can also publish a public event to the Pub/Sub system. Gateway takes care of authorization and authentication of WebSockets connection.

#### SNS/Firebase Cloud Messaging/etc.

Gateway product can push messages to mobile devices or other custom targets via existing cloud services. The difference between Pub/Sub connectors and endpoints is that connectors have low-level access to all events handled by Gateway (both public and private events). Endpoints are only means of transport for events pushed to the devices.

## Use Cases

### Modern web application

- frontend application running in a browser or mobile app,
- backend accessible via GraphQL or REST API,
- WebSockets support for reactivity

### Microservices

- many small functions
- functions deployed on different providers and available via different protocols (AWS Lambda, HTTP, GRPC) (functions discovery)
- sync/async communication between functions
- functions can be triggered from a legacy system
- react on custom events
- dynamically configure backend functions

### Data pipeline systems

- dynamically configure backend functions
- functions can be triggered from a legacy system
- react on custom events
- pull events from different systems (Kafka, RabbitMQ) via connectors

## Comparison

### Gateway vs FaaS providers

Gateway is NOT FaaS providers. It doesn't allow to deploy or call a function. Gateway integrates with existing FaaS providers (AWS Lambda, Google Cloud Functions, OpenWhisk Actions). Gateway features enable building large, serverless architectures in a unified way across different providers.

### Gateway vs OpenWhisk

Apache OpenWhisk is a integrate serverless platform. OpenWhisk is built around three concepts:
- actions
- triggers
- rules

OpenWhisk actions, as mentioned above, is a FaaS platform. Triggers & Rules enable building event-driven systems. Those two concepts are similar to Gateway's Pub/Sub system. Though, there are few differences:
- OpenWhisk Rules doesn't integrate with other FaaS provider
- OpenWhisk doesn't provide fine-grained ACL system
- OpenWhisk doesn't enable exporting events outside OpenWhisk

## Implementation

**Described Gateway product is a complex system. During the implementation phase, we need to make sure that every feature is requested by users and solves real users problems.**

Because of low latency & multi-region support requirements of some sub services Gateway instances has to be deployed in every supported region. Instances deployed in different regions and zones create a cluster. Gateway cluster uses consensus algorithm to prevent from data inconsistency between instances in a cluster.

```

          AWS us-east-1────────────────────┐
┌─────────┴─────┐                          │
│               │                          │
│    Gateway    │           ┌───┐          │
│   instance    │◀─────────▶│ λ ├┐         │
│               │           └┬──┘│         │
└─────────┬─────┘            └───┘         │
        ▲ └────────────────────────────────┘
        │                                   
        │                                   
        ▼ GCloud us-central1───────────────┐
┌─────────┴─────┐                          │
│               │                          │
│    Gateway    │           ┌───┐          │
│   instance    │◀─────────▶│ λ ├┐         │
│               │           └┬──┘│         │
└─────────┬─────┘            └───┘         │
        ▲ └────────────────────────────────┘
        │                                   
        │                                   
        │                                   
        ▼ Azure us-west-2──────────────────┐
┌─────────┴─────┐                          │
│               │                          │
│    Gateway    │           ┌───┐          │
│   instance    │◀─────────▶│ λ ├┐         │
│               │           └┬──┘│         │
└─────────┬─────┘            └───┘         │
          └────────────────────────────────┘
```

### Gateway & Platform

Platform is a SaaS product offered by Serverless, Inc. Platform is built on top of Gateway product. Platform provides UI and access management layer for Gateway.

```
Platform──────────────────────────────┐
│┌───────────────────────────────────┐│
││                UI                 ││
│└───────────────────────────────────┘│
│┌───────────────────────────────────┐│
││                                   ││
││                                   ││
││    Users/teams/orgs management    ││
││                                   ││
││                                   ││
│└───────────────────────────────────┘│
│┌───────────────────────────────────┐│
││                                   ││
││                                   ││
││              Gateway              ││
││                                   ││
││                                   ││
│└───────────────────────────────────┘│
└─────────────────────────────────────┘
```

### API

Gateway exposes configuration RESTful HTTP API.

#### Function discovery

##### Register function

`PUT /api/function`

Request:

- `id` - `string` - function name/ID e.g. `users/userCreate`
- `instances` - `array` of `object` - function instances:
  - `provider` - `string` - function provider, Possible values: `aws-lambda`, `gcloud-functions`, `openwhisk-action`
  - `originId` - `string` - provider specific ID
  - `region` - `string` - provider specific region name

##### Deregister function

`DELETE /api/function/<function id>`

#### Endpoints

##### Create endpoint

`POST /api/endpoint`

Request:

- `type` - `string` - endpoint type e.g. Possible values: `http`, `websockets` ...
- `target` - `array` of `object` - in case of `function` target type it's:
  - `functionId` - function id/name
  - `method` - HTTP method
  - `path` - URL path

Example:

```json
{
  "type": "http",
  "target": [{
    "functionId": "users/create",
    "method": "post",
    "path": "users"
  }]
}
```

Response:

- `id` - `string` - endpoint ID (something shorter than UUID as it will be a part of URL)
- `type` - ditto
- `endpoint` - `string` - in case of `http` type it's URL
- `target` - ditto

Example:

```json
{
  "id": "dogPzIz8",
  "endpoint": "https://gateway.internal/endpoint/dogPzIz8/",
  "type": "http",
  "target": [{
    "functionId": "users/create",
    "method": "post",
    "path": "users"
  }]
}
```

##### Delete endpoint

`DELETE /api/endpoint/<endpoint id>`