---
layout: post
title: Federated API Gateway using API Connect
tags: [APIConnect, automation]
---


![](https://higherlogicdownload.s3.amazonaws.com/IMWUC/DevCenterMigration/6e449490ad5b48f3a07088b7b16b592d_5-multi-cloud-gateways.png)

A customer asked me recently how to setup multiple API gateways work in different locations/configurations while still being controlled from a centralized API Management deployment. The architecture helps organizations in addressing multi-cloud or multi-datacenter challenges that serves various type of workloads, e.g., Internet workload or/and Interanet workload, and different type of configurations, e.g., v5-compatible or/and native-api-gateway.

In the following guide, I will share the steps required to setup API Connect deployment that uses two seperate API Gateway instances.

## Steps

### Install the Top Level APIConnectCluster CR

The Top Level custom resource will automatically deploy the following instances: API Manager, Developer Portal, API Analytics, API Gateway.

```yaml
apiVersion: apiconnect.ibm.com/v1beta1
kind: APIConnectCluster
metadata:
  annotations:
    apiconnect-operator/backups-not-configured: 'true'
  name: apic
  labels:
    app.kubernetes.io/instance: apiconnect
    app.kubernetes.io/managed-by: ibm-apiconnect
    app.kubernetes.io/name: apiconnect-small
spec:
  analytics:
    mtlsValidateClient: true
  imagePullSecrets:
    - ibm-entitlement-key
  imageRegistry: cp.icr.io/cp/apic
  license:
    accept: true
    license: L-MMBZ-295QZQ
    metric: VIRTUAL_PROCESSOR_CORE
    use: nonproduction
  portal:
    mtlsValidateClient: true
  profile: n1xc7.m48
  storageClassName: ocs-storagecluster-ceph-rbd
  version: 10.0.7.0
```

### Validate the API Connect is Ready

```bash
$ oc get apiconnectcluster
NAME   READY   STATUS   VERSION    RECONCILED VERSION   MESSAGE                        AGE
apic   6/6     Ready    10.0.7.0   10.0.7.0-5560        API Connect cluster is ready   17h
```
 
### Add a secondary API Gateway

* Fetech the `ingress` issurer, by running the following command:

```bash
$ oc get issuer | grep ingress
apic-ingress-issuer   True    17h
```

* Create the Certificates using the the `ingress` issurer

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: apic-gw-2-svc
  labels: {
    app.kubernetes.io/part-of: apic,
    app.kubernetes.io/name: "apic-gw-2-svc"
  }
spec:
  commonName: apic-gw-2-svc
  secretName: apic-gw-2-svc
  issuerRef:
    name: <apic-ingress-issuer>
  usages:
  - "client auth"
  - "signing"
  - "key encipherment"
  duration: 17520h
  renewBefore: 48h
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: apic-gw-2-peer
  labels: {
    app.kubernetes.io/part-of: apic,
    app.kubernetes.io/name: "apic-gw-2-peer"
  }
spec:
  commonName: apic-gw-2-peer
  secretName: apic-gw-2-peer
  issuerRef:
    name: <apic-ingress-issuer>
  usages:
  - "server auth"
  - "client auth"
  - "signing"
  - "key encipherment"
  duration: 17520h 
  renewBefore: 48h
  ```

* Create the admin password

```bash
 oc create secret generic apic-gw-2-admin --from-literal=password=supersecret
```


* Deploy the secondary API Gateway, and make sure you to update the following values: `apicGatewayPeeringTLS`, `apicGatewayPeeringTLS`, `issuers`, and `adminUser` to match your cluster configration.

```yaml
apiVersion: gateway.apiconnect.ibm.com/v1beta1
kind: GatewayCluster
metadata:
  annotations:
    apiconnect-operator/backups-not-configured: 'true'
  name: apic-gw-2
spec:
  peeringLogLevel: internal
  license:
    accept: true
    license: L-MMBZ-295QZQ
    metric: VIRTUAL_PROCESSOR_CORE
    use: nonproduction
  certManagerIssuer:
    kind: Issuer
    name: <apic-self-signed>
  imagePullSecrets:
    - ibm-entitlement-key
  configSequenceInterval: 3000
  apicGatewayPeeringTLS:
    secretName: apic-gw-2-peer
  apicGatewayServiceTLS:
    secretName: apic-gw-2-svc
  profile: n1xc1.m8
  webGUIManagementPort: 9090
  mtlsValidateClient: false
  microServiceSecurity: certManager
  apiDebugProbeMaxRecords: 1000
  apiDebugProbe: true
  webGUIManagementEnabled: false
  version: 10.0.7.0
  imageRegistry: cp.icr.io/cp/apic
  apiDebugProbeExpirationMinutes: 60
  gatewayEndpoint:
    annotations:
      cert-manager.io/issuer: <apic-ingress-issuer>
  apicGatewayServiceV5CompatibilityMode: false
  defaultLogFormat: text
  datapowerLogLevel: 3
  replicaCount: 1
  updateStrategy:
    mode: automatic
  gatewayManagerEndpoint:
    annotations:
      cert-manager.io/issuer: <apic-ingress-issuer>
  labels:
    app.kubernetes.io/part-of: apic
  adminUser:
    secretName: apic-gw-2-admin
```


### Validate the GatewayCluster are running

```bash
$ oc get gatewaycluster
NAME        READY   STATUS    VERSION    RECONCILED VERSION   AGE
apic-gw     2/2     Running   10.0.7.0   10.0.7.0-5560        17h
apic-gw-2   2/2     Running   10.0.7.0   10.0.7.0-5560        160m
```


### Register the secondary API Gateway to the API Connect Topology

Retrieve the newly addded endpoints to complate the API Gateway service registration: [Registering a gateway service](https://www.ibm.com/docs/en/api-connect/10.0.x?topic=topology-registering-gateway-service).

* Gateway Manager
```bash
$ oc get gatewaycluster apic-gw-2  -o=jsonpath='{.status.endpoints[0].uri}'
https://apic-gw-2-gateway-manager-[namespace].apps.[domain-name]
```

* API Gateway
```bash
$ oc get gatewaycluster apic-gw-2  -o=jsonpath='{.status.endpoints[1].uri}'
https://apic-gw-2-gateway-[namespace].apps.[domain-name]
```


---

To learn more, here are very good articles to read:
* [What Are API Gateways?](https://www.ibm.com/blog/api-gateway/)
* [Is API management a centralized or decentralized approach?](https://community.ibm.com/community/user/integration/blogs/kim-clark1/2018/12/10/is-api-management-a-centralized-or-decentralized-approach)


> The diagram credits goes to [Kim Clark](https://uk.linkedin.com/in/kimjulianclark).