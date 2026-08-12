# Kustomize workshop

```text
kubernetes/
├── development/
│   ├── deployment.yaml
│   └── service.yaml
├── staging/
│   ├── deployment.yaml
│   └── service.yaml
└── production/
    ├── deployment.yaml
    └── service.yaml
```

This structure deploys the same Inventory API to three environments. Each directory contains a complete copy of its Kubernetes manifests.

The image name is illustrative. This workshop renders YAML and does not deploy it.

## Duplicated manifests

* development
  * `kubernetes/development/deployment.yaml`

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: inventory-api
      labels:
        app.kubernetes.io/name: inventory-api
    spec:
      replicas: 1
      selector:
        matchLabels:
          app.kubernetes.io/name: inventory-api
      template:
        metadata:
          labels:
            app.kubernetes.io/name: inventory-api
        spec:
          containers:
            - name: inventory-api
              image: inventory-api:dev
              imagePullPolicy: IfNotPresent
              ports:
                - name: http
                  containerPort: 8080
              env:
                - name: LOG_LEVEL
                  value: DEBUG
              resources:
                limits:
                  memory: 256Mi
    ```

  * `kubernetes/development/service.yaml`

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: inventory-api
      labels:
        app.kubernetes.io/name: inventory-api
    spec:
      selector:
        app.kubernetes.io/name: inventory-api
      ports:
        - name: http
          port: 80
          targetPort: http
    ```

* staging
  * `kubernetes/staging/deployment.yaml`

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: inventory-api
      labels:
        app.kubernetes.io/name: inventory-api
    spec:
      replicas: 2
      selector:
        matchLabels:
          app.kubernetes.io/name: inventory-api
      template:
        metadata:
          labels:
            app.kubernetes.io/name: inventory-api
        spec:
          containers:
            - name: inventory-api
              image: inventory-api:rc
              imagePullPolicy: IfNotPresent
              ports:
                - name: http
                  containerPort: 8080
              env:
                - name: LOG_LEVEL
                  value: INFO
              resources:
                limits:
                  memory: 512Mi
    ```

  * `kubernetes/staging/service.yaml`

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: inventory-api
      labels:
        app.kubernetes.io/name: inventory-api
    spec:
      selector:
        app.kubernetes.io/name: inventory-api
      ports:
        - name: http
          port: 80
          targetPort: http
    ```

* production
  * `kubernetes/production/deployment.yaml`

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: inventory-api
      labels:
        app.kubernetes.io/name: inventory-api
    spec:
      replicas: 4
      selector:
        matchLabels:
          app.kubernetes.io/name: inventory-api
      template:
        metadata:
          labels:
            app.kubernetes.io/name: inventory-api
        spec:
          containers:
            - name: inventory-api
              image: inventory-api:1.0.0
              imagePullPolicy: IfNotPresent
              ports:
                - name: http
                  containerPort: 8080
              env:
                - name: LOG_LEVEL
                  value: WARN
              resources:
                limits:
                  memory: 1Gi
    ```

  * `kubernetes/production/service.yaml`

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: inventory-api
      labels:
        app.kubernetes.io/name: inventory-api
    spec:
      selector:
        app.kubernetes.io/name: inventory-api
      ports:
        - name: http
          port: 80
          targetPort: http
    ```

The Deployments differ only in these settings:

| Setting | Development | Staging | Production |
|---|---:|---:|---:|
| Replicas | 1 | 2 | 4 |
| Image tag | `dev` | `rc` | `1.0.0` |
| Log level | `DEBUG` | `INFO` | `WARN` |
| Memory | `256Mi` | `512Mi` | `1Gi` |

The Services are identical.

## Problems

* repeated changes
  * a shared change must be copied into every environment
* accidental environment drift
  * a copy may be missed or changed differently
* difficult reviews
  * reviewers must separate intended differences from duplicated YAML
* more duplication for every new environment
  * each environment adds another complete copy

## Kustomize structure

Kustomize replaces the copies with a shared base and environment overlays:

```text
kubernetes/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── development/
    ├── staging/
    └── production/
```

* resource
  * a Kubernetes manifest or Kustomize directory listed under `resources`
* base
  * a reusable Kustomize directory containing shared resources
  * a project convention, not a Kubernetes API object
  * composition-only in this workshop; render an overlay, not the base directly
* overlay
  * a Kustomize directory that references a base and describes one environment
* `images`
  * changes a matching container image name or tag
  * `name` selects the image value in the base; `newTag` supplies the environment tag
  * every overlay in this workshop sets `newTag`
* `replicas`
  * changes the replica count of a named workload
* `labels`
  * adds metadata used to identify and group resources
  * `includeTemplates: true` also labels Pods created by the Deployment
* `namespace`
  * assigns namespace-scoped resources to a namespace
  * does not create the Namespace object
* patch
  * changes fields that do not have a dedicated Kustomize transformer
  * [Kubernetes recommends small patches that do one thing](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#customizing)
  * a practical convention is one overlay patch per target resource
  * focused means a clear target and limited scope, not one file per changed field
* `imagePullPolicy`
  * controls when the kubelet asks the container runtime to pull an image
  * `Always` resolves the image through the registry whenever a container starts; cached layers are reused when the resolved digest is already present
  * `IfNotPresent` pulls the image only when it is absent from the node
  * `Never` uses only an image already present on the node and fails when it is absent
  * production should use an immutable tag or image digest rather than a mutable tag

## Workshop plan

1. Inspect the duplicated manifests.
   * Confirm that all Services are identical.
   * Identify the four intentional Deployment differences.
2. Extract the common resources.
   * Move one Deployment and one Service into `kubernetes/base`.
   * List both files as resources in the base `kustomization.yaml`.
3. Create the development overlay.
   * Reference the base.
   * Set the namespace, environment label, image tag, and replica count.
   * Patch only the development log level.
4. Create the staging overlay.
   * Set the staging namespace, label, image tag, and replica count.
   * Add the staging memory limit to its Deployment patch.
5. Create the production overlay.
   * Set the production namespace, label, image tag, and replica count.
   * Patch the production log level and memory limit.
6. Render every overlay.
   * Compare the generated resources with the required values.
   * Confirm that the shared Service is produced for every environment.
7. Handle a shared operational change.
   * Requirement: do not send Service traffic to an Inventory API Pod before `/ready` succeeds.
   * Add one readiness probe to the base Deployment instead of editing three copies.
   * Render every overlay and confirm that all inherit the probe.
8. Handle an environment exception.
   * Requirement: staging needs 15 seconds before its first readiness check.
   * Add `initialDelaySeconds` to the existing staging Deployment patch.
   * Confirm that development and production retain the shared five-second value.
9. Handle a mutable development image.
   * Requirement: the `dev` tag may point to a newly built image.
   * Add `imagePullPolicy: Always` to the existing development Deployment patch.
   * Keep `IfNotPresent` in staging and production, where tags are treated as immutable.

## Base

* `kubernetes/base/deployment.yaml`
  * contains the shared Deployment
  * uses the untagged `inventory-api` value as the match target for each overlay's `images` transformer
  * rendering the base directly would imply the `latest` tag; the overlays are the rendering targets
  * the readiness probe represents the shared operational change from step 7

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: inventory-api
    labels:
      app.kubernetes.io/name: inventory-api
  spec:
    replicas: 1
    selector:
      matchLabels:
        app.kubernetes.io/name: inventory-api
    template:
      metadata:
        labels:
          app.kubernetes.io/name: inventory-api
      spec:
        containers:
          - name: inventory-api
            image: inventory-api
            imagePullPolicy: IfNotPresent
            ports:
              - name: http
                containerPort: 8080
            readinessProbe:
              httpGet:
                path: /ready
                port: http
              initialDelaySeconds: 5
              periodSeconds: 10
            env:
              - name: LOG_LEVEL
                value: INFO
            resources:
              limits:
                memory: 256Mi
  ```

* `kubernetes/base/service.yaml`
  * contains the shared Service

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: inventory-api
    labels:
      app.kubernetes.io/name: inventory-api
  spec:
    selector:
      app.kubernetes.io/name: inventory-api
    ports:
      - name: http
        port: 80
        targetPort: http
  ```

* `kubernetes/base/kustomization.yaml`
  * includes both shared resources

  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - deployment.yaml
    - service.yaml
  ```

## Development overlay

* `kubernetes/overlays/development/kustomization.yaml`

  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  namespace: inventory-development
  resources:
    - ../../base
  images:
    - name: inventory-api
      newTag: dev
  replicas:
    - name: inventory-api
      count: 1
  labels:
    - pairs:
        app.kubernetes.io/environment: development
      includeTemplates: true
  patches:
    - path: deployment-patch.yaml
  ```

* `kubernetes/overlays/development/deployment-patch.yaml`
  * contains the development-specific changes for the Deployment
  * uses `Always` because the `dev` image tag is mutable

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: inventory-api
  spec:
    template:
      spec:
        containers:
          - name: inventory-api
            imagePullPolicy: Always
            env:
              - name: LOG_LEVEL
                value: DEBUG
  ```

## Staging overlay

* `kubernetes/overlays/staging/kustomization.yaml`

  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  namespace: inventory-staging
  resources:
    - ../../base
  images:
    - name: inventory-api
      newTag: rc
  replicas:
    - name: inventory-api
      count: 2
  labels:
    - pairs:
        app.kubernetes.io/environment: staging
      includeTemplates: true
  patches:
    - path: deployment-patch.yaml
  ```

* `kubernetes/overlays/staging/deployment-patch.yaml`
  * contains the staging-specific changes for the Deployment
  * inherits the readiness path, port, and period from the base

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: inventory-api
  spec:
    template:
      spec:
        containers:
          - name: inventory-api
            readinessProbe:
              initialDelaySeconds: 15
            resources:
              limits:
                memory: 512Mi
  ```

## Production overlay

* `kubernetes/overlays/production/kustomization.yaml`

  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  namespace: inventory-production
  resources:
    - ../../base
  images:
    - name: inventory-api
      newTag: "1.0.0"
  replicas:
    - name: inventory-api
      count: 4
  labels:
    - pairs:
        app.kubernetes.io/environment: production
      includeTemplates: true
  patches:
    - path: deployment-patch.yaml
  ```

* `kubernetes/overlays/production/deployment-patch.yaml`
  * changes the log level and memory limit

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: inventory-api
  spec:
    template:
      spec:
        containers:
          - name: inventory-api
            env:
              - name: LOG_LEVEL
                value: WARN
            resources:
              limits:
                memory: 1Gi
  ```

## Render overlays

* prerequisite
  * install `kubectl` with built-in Kustomize support
  * Docker Desktop and a Kubernetes cluster are not required for local rendering
* development

  ```bash
  kubectl kustomize kubernetes/overlays/development
  ```

* staging

  ```bash
  kubectl kustomize kubernetes/overlays/staging
  ```

* production

  ```bash
  kubectl kustomize kubernetes/overlays/production
  ```

Each command prints the final Service and Deployment manifests. It does not deploy them.
