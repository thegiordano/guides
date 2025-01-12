## Istio

### Overview

Istio is a service mesh that provides a dedicated layer to manage communication between microservices. It acts as a control plane for observability, security, traffic management, and resilience in distributed systems.

Core Concepts:
- Service Mesh: A network layer to control service-to-service communication in microservices architectures.
- Sidecar Proxy: Leverages Envoy to intercept and manage network traffic for services.

Benefits:
- Observability: Built-in monitoring, tracing, and logging (integrates with Prometheus, Grafana, Jaeger, etc.).
- Traffic Management: Advanced routing, load balancing, retries, and fault injection.
- Security: mTLS for secure communication, service authentication, and fine-grained access control.
- Resilience: Automatic retries, timeouts, and circuit breakers.
- Decoupled Logic: Offloads service discovery, load balancing, and policy enforcement from application code.

---
### Installation
Add the Istio Helm repository and update:
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

Install the CRDS:
```bash
helm install istio-base istio/base -n istio-system --create-namespace
```

Install Istio itself; set the values to use the Istio Ingress Gateway:
```bash
helm install istiod istio/istiod -n istio-system \
  --set meshConfig.ingressService=istio-gateway \
  --set meshConfig.ingressSelector=gateway
```

### Traffic management
---
#### Prerequisites
To route the traffic within the cluster leveraging Istio's advanced features, an Envoy sidecar must be injected into the Pods. This can either be configured
- on the namespace level with the label: ```istio-injection: enabled```
- on the pod level with the label: ```sidecar.istio.io/inject: "true"```
- manually via ```istioctl kube-inject -f DEPLOYMENT_NAME.yml```

---
#### DestinationRule
DestinationRule configures policies for traffic intended for a service after routing has occurred.
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: first-app
  namespace: staging
spec:
  host: first-app # service name that exposes 2 deployments
  subsets:
    - name: v1
      labels:
        app: first-app # selector for the pod defined in the service
        version: v1 # destionation pod unique label
    - name: v2
      labels:
        app: first-app
        version: v2
```

---
#### VirtualService
VirtualService Defines how requests to a service are routed within the mesh.
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: first-app
  namespace: staging
spec:
  hosts:
    - first-app
  http:
    - route:
        - destination:
            host: first-app
            subset: v1
          weight: 10 # 10% of the traffic goes to the v1 Subset defined in the DestinationRule
        - destination:
            host: first-app
            subset: v2
          weight: 90
```

---
#### Istio Gateway
*Gateway* describes a load balancer operating at the edge of the mesh receiving incoming or outgoing HTTP/TCP connections. The specification describes a set of ports that should be exposed, the type of protocol to use, SNI configuration for the load balancer, etc.
- SNI (Server Name Indication) is a TLS extension that enables clients to specify the hostname during the TLS handshake, ensuring the correct SSL certificate is presented before the HTTP connection opens.

Install the Helm chart; this will deploy a Service of type LoadBalancer (NLB on AWS):
```bash
helm install istio-ingress istio/gateway -n istio-ingress --create-namespace
```

The VirtualService now has to include the external DNS name; use the DNS name of the load balancer or provide your purchased domain name. You can also use a fake domain name, and simply edit your ```hosts``` file or append the fake domain name in the ehader of ```curl```.
```yaml
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: second-app
  namespace: production
spec:
  hosts:
    - app.my-domain.com # external domain name
    - second-app # kubernetes Service that exposes 2 deployments
  gateways:
    - api # specify the Istio Gateway resource
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            host: second-app
            subset: v1
          weight: 10
        - destination:
            host: second-app
            subset: v2
          weight: 90
```

Define an Istio Gateway to route external traffic to the two deployments that are exposed by the ```second-app``` Service internally.
```yaml
---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: api
  namespace: production
spec:
  selector:
    istio: gateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - app.my-domain.com
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - app.my-domain.com
      tls:
        credentialName: my-domain-com-crt # k8s secret holding the tls certificate for the domain name.
        mode: SIMPLE
```

---
### Sources
- https://istio.io/latest/docs/setup/install/helm/
- https://www.youtube.com/watch?v=H4YIKwAQMKk&t
- https://github.com/antonputra/tutorials/tree/main/lessons/155
- https://istio.io/latest/docs/reference/config/networking/gateway/
