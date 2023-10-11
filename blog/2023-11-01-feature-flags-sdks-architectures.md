---
slug: 'feature-flags-sdks-architectures'
title: 'feature-flags-sdks-architectures'
date: 2023-11-01
authors: [liran2000]
description: "Feature Flags services Server SDKs different architectures approaches"
tags: [client, sdk]
draft: false
---

Server SDKs (software development kits) are common for letting applications like microservices interact with Feature Flags services.  
Feature Flags SDKs has several used functionalities, mainly feature flag evaluation with or without a context. Each Feature Flags service can publish SDKs in multiple programming languages.
The Feature Flags services commonly expose APIs via endpoints like REST HTTP and/or gRpc.
SDKs are an important layer, as request-response loads affects both the application and the Feature Flags services.
When the Feature Flags service is Cloud service for all its users together, some services support a relay proxy which can be 
deployed on an organization network. The Relay Proxy lets multiple clients to connect to a local proxy, reducing number of 
outbound connections to the Cloud Service. Considering that a large amount of microservices using SDKs instances can be deployed, 
this can be significant.


<!--truncate-->

## Feature Flags services Server SDKs different architectures approaches

During usage and development involvement at several feature flag services providers, I have encountered some different 
architectures approaches used by some vendors.  
Here some of the main server-side SDKs functionalities is described, however, there are additional functionalities like 
statistics reporting, metrics, events and more.

### "Direct" API endpoints Bridge
At this approach, the Feature Flags service expose APIs via endpoints like REST HTTP and/or gRpc, and the SDK acts as like a bridge, 
executing the endpoint requests, and translate the request/responses to the programming language.  
Feature flag evaluation is done on proxy/service only.

###  "Direct" API endpoints Bridge SDK - example flow

```mermaid
sequenceDiagram
  participant SDK as SDK (server-side)
  participant features as Features Service
  SDK->>features: getBooleanEvaluation
  features->>SDK: true/false
  note left of SDK: another request
  SDK->>features: getBooleanEvaluation
  features->>SDK: true/false
```

#### Advantages
* Can be relatively simple to develop and maintain from the APIs, even by code generation.

#### Disadvantages
* Functional disadvantage is that intermediate issue like network or feature flags service error is handled by not evaluating
  to the expected feature flag value.
* Another functional disadvantage is that intermediate issue like network or feature flags service error during microservice 
  initialization is handled by not evaluating to the expected feature flag value.
* Performance and resources usage: each operation like feature flag evaluation causing network traffic. 
  * Time-consuming.
  * High load on network traffic.
  * Affecting both application, and Feature Flag service itself, whether it is self-hosted relay proxy or even more when it is a 
  * Cloud Saas service.

### API endpoints requests with cache
At this approach, the Feature Flags service expose APIs via endpoints like REST HTTP and/or gRpc, and the SDK calling the 
endpoint requests, and translate the request/responses to the programming language. A cache is used for responses data.  
Feature flag evaluation is done on proxy/service only.

###  API endpoints requests with cache - example flow

```mermaid
sequenceDiagram
  participant SDK as SDK (server-side)
  participant features as Features Service
  SDK->>features: getBooleanEvaluation
  features->>SDK: true/false
  note left of SDK: another request
  SDK->>SDK: getBooleanEvaluation true/false from cache
```

#### Advantages
* Network traffic can be reduced.

#### Disadvantages
* Functional disadvantage is that intermediate issue like network or feature flags service error for non-cached data is 
  handled by not evaluating to the expected feature flag value.
* Another functional disadvantage is that intermediate issue like network or feature flags service error during microservice
  initialization is handled by not evaluating to the expected feature flag value.
* First requests are not in cache, thus slower and causing network traffic.
* Not every request can be efficiently cached, as some evaluations can be dynamic according to context with different values.
* When there is no background task for getting real-time configuration changes event and refreshing the cache, 
  values can take time to be updated.

### Local evaluation
At this approach, the Feature Flags configuration is saved locally at the SDK.  
Feature flag evaluation is done on locally via the SDK and not affecting network traffic.
With this approach, configuration can be fetched on initialization, and be refreshed periodically and/or when there is 
a configuration change, which can be triggered at the SDK via stream event from the Feature Flags service.

###  Local evaluation - example flow

```mermaid
sequenceDiagram
  participant SDK as SDK (server-side)
  participant features as Features Service
  participant developer as App team
  SDK->>features: fetch config https://...
  note left of SDK: save config in cache
  SDK->>features: <stream> fetch config changes events https://...
  features->>SDK: OK
  note left of features: fetch config changes responses
  SDK->>SDK: getBooleanEvaluation true/false, evaluated locally from cached config
  developer->>features: Change feature flag value
  features->>SDK: Changed event
  SDK->>features: fetch config https://...
  features->>SDK: Error
  SDK->>SDK: getBooleanEvaluation true/false, evaluated locally from cached config
```

#### Advantages
* Functional advantage is that intermediate issue like network or feature flags service error is automatically handled, as 
  configuration is cached, and flags are evaluated locally.
  If an issue like network error is ongoing for a long time period, SDK provider state can also be reflected as stale.
* Another functional advantage, initialization can have a fallback mechanism to dealing with situations like temporary network failure by starting from last 
  received configuration or some other configuration, thus microservice can start with last feature flag configuration.
* Feature flag evaluation is done on locally via the SDK and not affecting network traffic.

#### Disadvantages
* When feature flags configuration is very large with many feature flags, network traffic and 
  local cache sizes can be increased.
* Local evaluation should be implemented at the SDK, considering rules like context properties based and percentage rollout.



## OpenFeature and server-side SDKs
* OpenFeature has [specification](https://openfeature.dev/specification/glossary/#feature-flag-sdk) for SDKs for 
  multiple programming languages for developing a unified API and SDK.
* OpenFeature specification has methods like flags evaluation and initialization.
* OpenFeature specification treats events, which can go together with scenarios like network errors.
* As there are multiple SDKs architectures approaches, fallback mechanisms are not always well-defined.


