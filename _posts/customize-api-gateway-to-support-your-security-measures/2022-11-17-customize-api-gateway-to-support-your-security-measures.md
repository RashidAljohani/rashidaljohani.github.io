---
layout: post
title: Customize API Gateway to Support your Security Measures
tags: [api, security, datapower]
---


![](./banner.png)

A friend of mine asked me on another day .. how he could help his customers to modify API Connect configuration to support a special case that was raised by the security team. API Connect uses DataPower as API Gateway to support securing and managing the APIs traffic. Part of that is allowing end-users to define their own policies besides the [built-in](https://www.ibm.com/docs/en/api-connect/10.0.5.x_lts?topic=constructs-built-in-policies) ones. 

In the following steps, you will learn how to define a global security policy that removes a particular key from the message's headers as a post-response policy for a gateway service.


* Define the policy logic, and save it to `remove-x-client-ip.yaml`

```yaml
global-policy: 1.0.0
info:
  name: remove-x-client-ip
  title: remove-x-client-ip
  version: 1.0.0
gateways:
  - datapower-api-gateway
assembly:
  execute:
      - set-variable:
          version: 2.0.0
          title: Remove X-Client-IP
          actions:
            - clear: message.headers.X-Client-IP
```

You will notice that I've used one of the built-in policy that DataPower provide: [set-variable](https://www.ibm.com/docs/en/api-connect/10.0.5.x_lts?topic=policies-set-variable) to clearn a key from the message's headers.


* Login to API Manager:

```bash
$ apic login --server <apim-endpoint>
```

* Create a global policy:

```bash
$ apic global-policies:create --catalog sandbox --configured-gateway-service <api-gateway-service-name> --org <porg-name> --server <apim-endpoint> --scope catalog remove-x-client-ip.yaml
```

* List global policies to validate:

```bash
$ apic global-policies:list-all --catalog sandbox --configured-gateway-service <api-gateway-service-name> --org <porg-name> --server <apim-endpoint> --scope catalog
```

* Write the URL of the required global policy to a `.yaml` file:

```bash
$ apic global-policies:get --catalog sandbox --configured-gateway-service <api-gateway-service-name> --org <porg-name>--server <apim-endpoint> --scope catalog remove-x-client-ip:1.0.0 --fields url
```

* Edit the GlobalPolicy.yaml file and replace the string `url` with `global_policy_url`. The resulting file has the following format:

```bash
global_policy_url: >-
  https://server_host_name/api/catalogs/catalog_id/configured-gateway-services/gateway_service_id/global-policies/policy_id
```

* In this case, we want to designate the global policy to be the [post-response](https://www.ibm.com/docs/en/api-connect/10.0.5.x_lts?topic=applications-working-global-policies) policy:

```bash
$ apic global-policy-posthooks:create --catalog sandbox --configured-gateway-service <api-gateway-service-name> --org <porg-name> --server <apim-endpoint> --scope catalog GlobalPolicy.yaml
```


* Test your APIs to validate the policy

```yaml
--
headers: Object
    accept: "application/json"
    apim-debug: "true"
    user-agent: "*******"
    sec-ch-ua-platform: "*******"
    origin: "https://*******.us-south.containers.appdomain.cloud"
    sec-fetch-mode: "cors"
    sec-fetch-dest: "empty"
    referer: "https://*******.us-south.containers.appdomain.cloud/"
    accept-encoding: "*******"
    content-type: "application/json"
    author: "CSM"
    Access-Control-Allow-Origin: "https://*******.us-south.containers.appdomain.cloud"
    Access-Control-Allow-Credentials: "true"
Vary: "Origin"
Date: "Thu, 17 Nov 2022 11:45:51 GMT"
body: "{"response":{"code":200,"message":"Hello World!"}}"
--
```
