# kustomize-workshop

## References

* [Declarative management of Kubernetes objects using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
* [`kubectl kustomize` command reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/)
* [Kubernetes container images](https://kubernetes.io/docs/concepts/containers/images/)
* [Kubernetes liveness, readiness, and startup probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/)

## Kustomize

* purpose
  * customizes Kubernetes resources without introducing a template language
  * keeps common configuration in one place and expresses intentional differences separately
  * is built into `kubectl`; a separate Kustomize installation is not required for this workshop
* input
  * starts from regular Kubernetes YAML rather than manifests containing placeholders
  * reads build instructions from a `kustomization.yaml` file
  * composes resources from files and other Kustomize directories
* build process
  * loads the resources listed by the selected `kustomization.yaml`
  * recursively loads referenced bases
  * applies built-in transformations such as namespaces, labels, images, and replicas
  * applies patches for targeted resource changes
  * emits complete Kubernetes manifests
* source and output
  * does not modify the source manifests while rendering
  * does not contact a cluster or deploy resources when using `kubectl kustomize`
  * produces YAML that can be reviewed, compared, or passed to a separate deployment step
* benefits
  * shared changes are made once in the base
  * overlays contain only environment-specific intent
  * rendered output remains standard Kubernetes YAML
* limits
  * does not provide empty or required-value placeholders
  * transformations need existing resource names or image names to select their targets
  * repository conventions or external validation must ensure every required overlay value is set

## Problems Kustomize addresses

* duplicated manifests
  * shared configuration is defined once and reused
* accidental environment drift
  * overlays expose intentional differences
* difficult reviews
  * reviews focus on small environment-specific changes
* environment growth
  * new environments reuse existing resources
* example
  * requirement: add the same readiness probe to every environment
  * without Kustomize
    * repeat this block in three Deployment files

      ```yaml
      readinessProbe:
        httpGet:
          path: /ready
          port: http
      ```

  * with Kustomize
    * add the block once to `base/deployment.yaml`
    * development, staging, and production inherit it

## example
* structure
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
