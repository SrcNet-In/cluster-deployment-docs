# Gateway API

## Premise
To be generally available, web applications running on Kubernetes need to be exposed outside the cluster. This can be achieved in a number of ways.

The easiest way to expose a service or API endpoint outside a Kubernetes cluster is to assign a service type of `NodePort`. The drawback of this approach is that the service runs on a non-standard port number. Also, each cluster has a finite number of ports assigned to its NodePort pool.

These limitations can be overcome by using either the **Ingress API** or the **Gateway API**. While the Ingress API has been widely deployed in the Kubernetes world, it has its own drawbacks. Notably, a strong reliance on annotations in Ingress manifests makes the Ingress API inflexible and difficult to write generic templates for in Helm charts.

The **Gateway API** overcomes these drawbacks by providing a **generic, easy to templatise, and extensible** way to define resources which provide an ingress of traffic to Kubernetes-hosted endpoints.

In the following sections, we will compare the two approaches.

---

## Ingress API

HTTP and HTTPS network services may be exposed outside the Kubernetes cluster using Ingress resources, provided as part of the Ingress API. Traffic is routed using rules defined within the Ingress resource.

Before Ingress resources can be defined, the cluster must have at least one Ingress Controller running. This Ingress Controller service is typically exposed using a `LoadBalancer` service type.

The Ingress API is widely adopted by Kubernetes users and well-supported by vendors, with many implementations (Ingress controllers) available.

### Ingress API resource organisation

![ingress architecture](https://kubernetes.io/docs/images/ingress.svg)

### Limitations of the Ingress API

- **Limited features**: The Ingress API only supports TLS termination and simple content-based request routing of HTTP traffic.
- **Reliance on annotations for extensibility**: The annotations approach leads to limited portability as every implementation has its own supported extensions.
- **Insufficient permission model**: The Ingress API is not well-suited for multi-team clusters with shared load-balancing infrastructure.

---

## Gateway API

The **Gateway API** is an official Kubernetes project being worked on by the Kubernetes Network SIG, representing the next generation of Ingress, Load Balancing, and Service Mesh APIs. It focuses on **L4 and L7 routing** within Kubernetes. It is designed to be **generic, expressive, and role-oriented**.

### Features of the Gateway API

The following design goals drive the concepts of Gateway API and demonstrate how Gateway aims to improve upon current standards like Ingress:

- **Role-oriented**: Gateway is composed of API resources which model organizational roles that use and configure Kubernetes service networking.
- **Portable**: Like Ingress, Gateway API is designed to be a portable specification supported by many implementations.
- **Expressive**: Gateway API supports features like header-based matching, traffic weighting, and others that were only possible in Ingress through custom annotations.
- **Extensible**: Allows for custom resources to be linked at various layers of the API for granular customization.

### Gateway API Resource Model

- **GatewayClass**: Cluster-scoped resource defining a set of Gateways that share a common configuration and behavior. Handled by a Gateway Controller.
- **Gateway**: Describes how traffic can be translated within a cluster. Acts as a gateway between external and internal traffic.
- **Route Resources**: Protocol-specific rules for mapping requests from a Gateway to Kubernetes Services:
  - `HTTPRoute`: Multiplexing HTTP or terminated HTTPS connections.
  - `GRPCRoute`: Routing gRPC traffic.
  - `TLSRoute` *(experimental)*: Multiplexing TLS connections via SNI.
  - `TCPRoute` and `UDPRoute` *(experimental)*: Mapping TCP/UDP ports to backends. May terminate TLS where appropriate.

### Gateway API resource organisation

Reference:  
![resource model](https://gateway-api.sigs.k8s.io/images/resource-model.png)

---

## Examples

The following example snippets illustrate the differences between Ingress API and Gateway API YAML manifests. Both create traffic routes to the same Service resource.

> **Note**: Installation of an Ingress Controller and Gateway Controller are outside the scope of this document.

---

### Ingress API

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.12.0-beta.0
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: science-portal-ingress
  namespace: skaha-system
  annotations:
    spec.ingressClassName: nginx
spec:
  ingressClassName: nginx
  rules:
  - host: canfar.e4r.internal
    http:
      paths:
      - path: /science-portal
        pathType: Prefix
        backend:
          service:
            name: science-portal-tomcat-svc
            port:
              number: 8080
```

---

### Gateway API

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway-class
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: envoy-canfar-gateway
spec:
  gatewayClassName: envoy-gateway-class
  listeners:
  - name: canfar-gateway-http-envoy
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All 
  - name: canfar-gateway-https-envoy
    protocol: HTTPS
    port: 443
    allowedRoutes:
      namespaces:
        from: All 
    hostname: "canfar.e4r.internal"
    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: canfar-gateway-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: envoy-canfar-gateway-httproutes
  namespace: skaha-system
spec:
  hostnames:
    - "canfar.e4r.internal"
  parentRefs:
    - name: envoy-canfar-gateway
      namespace: default
  rules:
    - backendRefs:
        - name: science-portal-tomcat-svc
          port: 8080
      matches:
        - path:
            type: PathPrefix
            value: /science-portal
```

---

## Example for Canfar deployment using Gateway API

The following snippet is an example to deploy **Canfar using Gateway API**.

### Step 1: Create a GatewayClass

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway-class
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

> The `controllerName` refers to the controller that will manage Gateways of this class.

---

### Step 2: Create a Gateway for Canfar

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: envoy-canfar-gateway
spec:
  gatewayClassName: envoy-gateway-class
  listeners:
  - name: canfar-gateway-http-envoy
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
  - name: canfar-gateway-https-envoy
    protocol: HTTPS
    port: 443
    allowedRoutes:
      namespaces:
        from: All
    hostname: "canfar.e4r.internal"
    tls:
      certificateRefs:
        - kind: Secret
          group: ""
          name: canfar-gateway-tls
```

---

### Step 3: Create an HTTPRoute for services in Canfar

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: envoy-canfar-gateway-httproutes
  namespace: skaha-system
spec:
  hostnames:
    - "canfar.e4r.internal"
  parentRefs:
    - name: envoy-canfar-gateway
      namespace: default
  rules:
    - backendRefs:
        - name: gms
          port: 8080
      matches:
        - path:
            type: PathPrefix
            value: /gms
    - backendRefs:
        - name: reg
          port: 8080
      matches:
        - path:
            type: PathPrefix
            value: /reg
    - backendRefs:
        - name: science-portal-tomcat-svc
          port: 8080
      matches:
        - path:
            type: PathPrefix
            value: /science-portal
    - backendRefs:
        - name: skaha-tomcat-svc
          port: 8080
      matches:
        - path:
            type: PathPrefix
            value: /skaha
```
---
### STEP 4: Create a Gateway for Harbor and attach the GatewayClass created in STEP 1.
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: envoy-harbor-gateway
spec:
  gatewayClassName: envoy-gateway-class
  # The same GatewayClass is used for Harbor.
  listeners:
  - name: harbor-gateway-http-envoy
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
    # This listener will accept HTTP traffic on port 80 from any namespace.

  - name: harbor-gateway-https-envoy
    protocol: HTTPS
    port: 443
    allowedRoutes:
      namespaces:
        from: All
    # This listener will accept HTTPS traffic on port 443 from any namespace.
    hostname: "harbor.e4r.internal"
    # Define a hostname for the Harbor gateway.
    tls:
      certificateRefs:
        - kind: Secret
          group: ""
          name: harbor-gateway-tls
        # The TLS configuration references a Secret containing the TLS certificate.
```
---
### STEP 5: Create an HTTPRoute for the services in Harbor and attach it to the Gateway created in STEP 4.
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: harbor-httproutes
  namespace: harbor
spec:
  hostnames:
    - "harbor.e4r.internal"
    # HTTPRoute applies to traffic matching this hostname.
  parentRefs:
    - name: envoy-harbor-gateway
      namespace: default
    # This HTTPRoute is attached to the 'envoy-harbor-gateway' Gateway defined earlier.
  rules:
    # Define routing rules for various paths within Harbor
    - backendRefs:
        - name: harbor-portal
          port: 80
      matches:
        - path:
            type: PathPrefix
            value: /
    - backendRefs:
        - name: harbor-core
          port: 80
      matches:
        - path:
            type: PathPrefix
            value: /c
    - backendRefs:
        - name: harbor-core
          port: 80
      matches:
        - path:
            type: PathPrefix
            value: /chartrepo
    - backendRefs:
        - name: harbor-core
          port: 80
      matches:
        - path:
            type: PathPrefix
            value: /v2
    - backendRefs:
        - name: harbor-core
          port: 80
      matches:
        - path:
            type: PathPrefix
            value: /service
    - backendRefs:
        - name: harbor-core
          port: 80
      matches:
        - path:
            type: PathPrefix
            value: /api
```

## Reference Resources
- [https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/#reasons-to-switch-to-gateway-api](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/#reasons-to-switch-to-gateway-api)
- [https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api](https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api)
- [https://gateway-api.sigs.k8s.io/guides/simple-gateway/](https://gateway-api.sigs.k8s.io/guides/simple-gateway/)