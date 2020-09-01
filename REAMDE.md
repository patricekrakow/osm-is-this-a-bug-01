# OSM | Is this a BUG?

## Scenario

### Step 1 - Initial Situation without the Mesh

```text
client-u-pod --> 200 { get, alpha-api, /path-01 } | service-v-pod
```

### Step 2 - Add the Mesh

```text
client-u-pod --> 404 { get, alpha-api, /path-01 } | proxy | service-v-pod
```

### Step 3 - Configure the Access Control

```text
client-u-pod --> 200 { get, alpha-api, /path-01 } | proxy | service-v-pod
```

### Step 4 - Add a new microservice with the same interface

```text
client-u-pod --> 200 { get, alpha-api, /path-01 } | proxy | service-v-pod
             --> 404 { get, alpha-api, /path-01 } | proxy | service-w-pod
```

## Demo

### Step 1 - Initial Situation without the Mesh

We have

```text
client-u-pod --> 200 { get, alpha-api, /path-01 } | service-v-pod
```

where

* the _Pod_ `client-u-pod` is using the _ServiceAccount_ `client-u`
* the _Pod_ `service-v-pod` is using the _ServiceAccount_ `service-v`
* the _Pod_ `service-v-pod` is reachable via the _Service_ `alpha-api`

```text
$ kubectl apply -f mfst-1.yaml
...
```

<details><summary>Click to expand <code>mfst-1.yaml</code></summary>

```yaml
# mfst-1.yaml
---
# Deploy 'demo' Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
# Deploy 'service-v' Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-v
  namespace: demo
---
# Deploy 'service-v-deployment' Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-v-deployment
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-v
      api: alpha-api
  template:
    metadata:
      labels:
        app: service-v
        api: alpha-api
    spec:
      serviceAccountName: service-v
      containers:
      - name: service-v
        image: patrice1972/service-v:1.0.0
        ports:
        - name: service-v-port
          protocol: TCP
          containerPort: 3000
---
# Deploy 'service-v' Service
apiVersion: v1
kind: Service
metadata:
  name: service-v
  namespace: demo
spec:
  selector:
    app: service-v
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
---
# Deploy 'alpha-api' Service
apiVersion: v1
kind: Service
metadata:
  name: alpha-api
  namespace: demo
spec:
  selector:
    api: alpha-api
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
---
# Deploy 'client-u' Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: client-u
  namespace: demo
---
# Deploy 'client-u-deployment' Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-u-deployment
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client-u
  template:
    metadata:
      labels:
        app: client-u
    spec:
      serviceAccountName: client-u
      containers:
      - name: client-u
        image: patrice1972/client-u:1.0.0
        env:
        - name: API_HOST
          value: "alpha-api.demo"
---
# Deploy 'client-u' (dummy) Service [Required by OSM]
apiVersion: v1
kind: Service
metadata:
  name: client-u
  namespace: demo
spec:
  selector:
    app: client-u
  ports:
  - protocol: TCP
    port: 9999
    name: dummy-unused-port
```

</details>

```text
$ kubectl get pods -n demo
...
$ kubectl logs ... -n demo | tail
...
```

### Step 2 - Add the Mesh

We will then have

```text
client-u-pod --> 404 { get, alpha-api, /path-01 } | proxy | service-v-pod
```

```text
$ export PATH="$HOME/environment/go/src/github.com/openservicemesh/osm/bin:$PATH"
$ osm version
Version: dev; Commit: 930991cbc4a211c43c15195a7a4c7d2c867ddf05; Date: 2020-08-25-09:22
$ osm install --osm-image-tag 930991cbc4a211c43c15195a7a4c7d2c867ddf05
OSM installed successfully in namespace [osm-system] with mesh name [osm]
$ osm namespace add demo
Namespace [demo] successfully added to mesh [osm]
$ kubectl delete pods --all -n demo
pod "client-x-..." deleted
pod "service-a-..." deleted
$ kubectl get pods -n demo
...
$ kubectl logs ... -n client-x demo | tail
...
```

### Step 3 - Configure the Access Control

We will then have

```text
client-u-pod --> 200 { get, alpha-api, /path-01 } | proxy | service-v-pod
```

```text
$ kubectl apply -f mfst-2.yaml
...
```

<details><summary>Click to expand <code>mfst-2.yaml</code></summary>

```yaml
# mfst-2.yaml
---
apiVersion: specs.smi-spec.io/v1alpha3
kind: HTTPRouteGroup
metadata:
  name: alpha-api-routes
  namespace: demo
spec:
  matches:
  - name: get-path-01
    pathRegex: /path-01
    methods:
    - GET
---
# Deploy the 'allow-client-u-to-service-v-through-alpha-api-routes' TrafficTarget
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha2
metadata:
  name: allow-client-u-to-service-v-through-alpha-api-routes
  namespace: demo
spec:
  destination:
    kind: ServiceAccount
    name: service-v
    namespace: demo
    port: 3000
  rules:
  - kind: HTTPRouteGroup
    name: alpha-api-routes
    matches:
    - get-path-01
  sources:
  - kind: ServiceAccount
    name: client-u
    namespace: demo
```

</details>

```text
$ kubectl get pods -n demo
...
$ kubectl logs ... -n demo | tail
...
```

### Step 4 - Add a new microservice with the same interface

We should then have

```text
client-u-pod --> 200 { get, alpha-api, /path-01 } | proxy | service-v-pod
             --> 404 { get, alpha-api, /path-01 } | proxy | service-w-pod
```

```text
$ kubectl apply -f mfst-3.yaml
...
```

<details><summary>Click to expand <code>mfst-3.yaml</code></summary>

```yaml
# mfst-3.yaml
---
# Deploy 'service-w' Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-w
  namespace: demo
---
# Deploy 'service-w-deployment' Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-w-deployment
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-w
      api: alpha-api
  template:
    metadata:
      labels:
        app: service-w
        api: alpha-api
    spec:
      serviceAccountName: service-w
      containers:
      - name: service-w
        image: patrice1972/service-w:1.0.0
        ports:
        - name: service-w-port
          protocol: TCP
          containerPort: 3000
---
# Deploy 'service-w' Service
apiVersion: v1
kind: Service
metadata:
  name: service-w
  namespace: demo
spec:
  selector:
    app: service-w
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
```

</details>

```text
$ kubectl get pods -n demo
...
$ kubectl logs ... -n demo | tail
...
```
