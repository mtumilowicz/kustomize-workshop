# kustomize-workshop

## References

* [Declarative management of Kubernetes objects using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
* [`kubectl kustomize` command reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/)
* [Kubernetes container images](https://kubernetes.io/docs/concepts/containers/images/)
* [Kubernetes liveness, readiness, and startup probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/)
* [Helm introduction](https://helm.sh/docs/intro/introduction/)
* [Helm chart template guide](https://helm.sh/docs/chart_template_guide/)
* [Helm chart structure](https://helm.sh/docs/topics/charts/)
* [Helm values best practices](https://helm.sh/docs/chart_best_practices/values/)
* [Helm chart dependencies](https://helm.sh/docs/chart_best_practices/dependencies/)
* [Helm post-rendering](https://helm.sh/docs/topics/advanced/#post-rendering)
* [Flux Kustomization](https://fluxcd.io/flux/components/kustomize/kustomizations/)
* [Flux Helm Controller](https://fluxcd.io/flux/components/helm/)

## Kustomize

* purpose
  * customizes Kubernetes resources without introducing a template language
  * keeps common configuration in one place and expresses intentional differences separately
  * is built into `kubectl`; a separate Kustomize installation is not required for this workshop
* input
  * starts from regular Kubernetes YAML rather than manifests containing placeholders
  * reads build instructions from a `kustomization.yaml` file
  * composes resources from files and other Kustomize directories
* configuration model
  * resource
    * a Kubernetes manifest or Kustomize directory listed under `resources`
  * example

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
    * base
      * a reusable Kustomize directory containing shared resources
      * a project convention, not a Kubernetes API object
      * composition-only in this workshop; render an overlay, not the base directly
    * overlay
      * a Kustomize directory that references a base and describes one environment  

* build process
  * loads the resources listed by the `kustomization.yaml` in the directory passed to `kubectl kustomize`
  * recursively loads referenced Kustomization directories and the resources they declare
  * applies built-in transformations such as namespaces, labels, images, and replicas
    * example: selects image tag
  * applies patches for targeted resource changes
    * example: deployment-patch.yaml 
  * emits complete Kubernetes manifests
* resource loading
  * Kustomize does not automatically load every YAML file in a directory
    * explicit references make builds deterministic; unrelated YAML files are not included accidentally
    * the file's role is declared in `kustomization.yaml`, not inferred from its directory or filename
        * example: `deployment-patch.yaml` is read as a patch because it is listed under `patches`; it is not emitted as a separate Deployment
  * `resources` explicitly defines the files and Kustomize directories that belong to a build
    * a referenced YAML file contributes its Kubernetes object
    * a referenced directory contributes the objects declared by its own `kustomization.yaml`
  * the loading chain is

    ```text
    overlays/development/kustomization.yaml # entry point passed to kubectl kustomize
      -> referenced kustomization.yaml       # Kustomization listed under resources
        -> referenced manifest files     # example: deployment.yaml, service.yaml
          > loaded Kubernetes objects # objects available for transformation, example: object from deployment.yaml
    ```

  * example
    * the development overlay references the base

      ```yaml
      # kubernetes/overlays/development/kustomization.yaml
      resources:
        - ../../base
      ```

    * Kustomize opens the referenced base and follows its resource declarations

      ```yaml
      # kubernetes/base/kustomization.yaml
      resources:
        - deployment.yaml
        - service.yaml
      ```

    * without those two base entries, `deployment.yaml` and `service.yaml` would be ignored
        * => overlay would have no Deployment or Service to transform
* transformations
  * a transformer is Kustomize processing logic, not a Kubernetes resource
  * it changes applicable fields in the loaded resources without requiring a patch
  * a transformation may affect one resource or many resources, depending on how it selects them
    * example
        * `namespace` assigns namespace to every namespace-scoped resource (Deployment, Service etc)
        * `labels` adds the environment label to loaded resources
        * `replicas` sets the replica count on the specified workload
  * example

    ```yaml
    # kubernetes/overlays/development/kustomization.yaml
    namespace: inventory-development # assigns the namespace
    ```

* patch targeting
  * a patch describes changes to one or more resources already loaded by the current build
    * referenced in `kustomization.yaml`
        ```
        patches:
          - path: deployment-patch.yaml
        ```
  * Kustomize searches only the resources already loaded by the current build
  * matching by
    * patch's resource identity
        * identifies its target by Kubernetes resource identity: API group/version, kind, and `metadata.name`
        * example
            ```
            # kustomization.yaml
            patches:
              - path: deployment-patch.yaml
          
            # deployment-patch.yaml
            apiVersion: apps/v1
            kind: Deployment
            ```
    * explicit `target` in `kustomization.yaml`
        * identifies its target by explicitly stated target
        * example
          ```yaml
          patches:
            - path: deployment-patch.yaml
              target: # every declared condition must match
                group: apps
                version: v1
                kind: Deployment
                name: inventory-api
                namespace: inventory-development
                labelSelector: app.kubernetes.io/name=inventory-api
          ```

  * [Kubernetes recommends small patches that do one thing](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#customizing)
  * a practical convention is one overlay patch per target resource
    * focused means a clear target and limited scope, not one file per changed field
  * example
    * the development overlay registers one patch

      ```yaml
      # kubernetes/overlays/development/kustomization.yaml
      patches:
        - path: deployment-patch.yaml
      ```

    * the patch content identifies its target

      ```yaml
      # kubernetes/overlays/development/deployment-patch.yaml
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

    * the identity must match a loaded resource

      | Patch field | Required target value |
      |---|---|
      | `apiVersion` | `apps/v1` |
      | `kind` | `Deployment` |
      | `metadata.name` | `inventory-api` |

    * Kustomize finds `Deployment/inventory-api` from `base/deployment.yaml` and merges the patch into it
    * `containers[].name: inventory-api` selects the Inventory API container inside that Deployment
    * `deployment-patch.yaml` and `deployment.yaml` could be renamed without changing the target, provided their references were updated
    * another Deployment with a different name is unaffected; a Deployment from an unreferenced base is never loaded
    * if the patch named `another-api`, rendering would fail because no matching target exists
* source and output
  * does not modify the source manifests while rendering
  * does not contact a cluster or deploy resources when using `kubectl kustomize`
  * produces YAML that can be reviewed, compared, or passed to a separate deployment step
* rendering
  * point `kubectl kustomize` to a directory containing `kustomization.yaml`

    ```bash
    kubectl kustomize kubernetes/overlays/development
    ```

  * prints the final Kubernetes YAML to standard output
  * does not deploy resources or require a cluster
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

## Kustomize and Helm

* purpose
  * Kustomize customizes Kubernetes resources for different targets
  * Helm packages, distributes, installs, and upgrades Kubernetes applications as charts and releases
* source format
  * Kustomize starts from Kubernetes resource YAML and applies transformations and patches
  * Helm charts use Go templates and values to generate Kubernetes resource YAML
* configuration
  * Kustomize expresses variants with bases and overlays
  * Helm exposes chart-defined settings through values
* lifecycle
  * `kubectl kustomize` renders manifests and does not track releases
  * Helm tracks each installed chart instance as a release and supports upgrades and rollbacks
* internal services with Helm
  * packaging company-owned services as Helm charts remains a supported approach
  * deployment as a product
    * does not mean that the application is sold externally
    * means that deployment behavior has a supported interface instead of requiring consumers to edit templates
    * example
      * the application team deploys the Inventory API to the company's cloud environments
      * customer operations teams deploy the same Inventory API in their on-premises Kubernetes clusters
      * the chart provides both groups with one supported deployment package and configuration interface
  * repository structure
    * Helm does not prescribe where environment value files must live
    * one clear repository convention is

      ```text
      deploy/
      ├── chart/
      │   └── inventory-api/
      │       ├── Chart.yaml
      │       ├── values.yaml
      │       ├── values.schema.json
      │       └── templates/
      │           ├── deployment.yaml
      │           └── service.yaml
      └── environments/
          ├── development.yaml
          ├── staging.yaml
          └── production.yaml
      ```

    * `chart/inventory-api/values.yaml` contains documented defaults
    * `environments/*.yaml` contains environment overrides
    * `values.schema.json` validates supported value types and constraints
    * `templates/` converts the values into Kubernetes resources
    * render production without installing it

      ```bash
      helm template inventory-api deploy/chart/inventory-api \
        --values deploy/environments/production.yaml
      ```

    * install or upgrade production

      ```bash
      helm upgrade --install inventory-api deploy/chart/inventory-api \
        --namespace inventory-production \
        --values deploy/environments/production.yaml
      ```

  * configuration interface
    * chart values are not automatically mapped to Kubernetes fields
    * a template must explicitly use each supported value
    * the Deployment template reads the supported replica and image values

      ```yaml
      # deploy/chart/inventory-api/templates/deployment.yaml
      spec:
        replicas: {{ .Values.replicaCount }}
        template:
          spec:
            containers:
              - name: inventory-api
                image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
      ```

    * `values.yaml` documents every supported property and supplies defaults

      ```yaml
      # replicaCount controls the number of Inventory API Pods.
      replicaCount: 1

      image:
        # image.repository identifies the Inventory API image.
        repository: inventory-api
        # image.tag selects the application version.
        tag: dev
      ```

    * `values.schema.json` can reject missing values, invalid types, or unsupported values during linting and rendering
  * reusable deployment behavior
    * a chart can standardize labels, security contexts, readiness probes, ServiceAccount creation, resource configuration, rollout strategy, and monitoring integration
    * environment files supply only the supported differences

      ```yaml
      image:
        repository: registry.example.com/inventory-api
        tag: 1.4.0

      replicas: 4

      resources:
        limits:
          memory: 1Gi
      ```

  * versioned package
    * `version` identifies the chart package and its deployment behavior
    * `appVersion` describes the application version represented by the chart and is independent of the chart version

      ```yaml
      # Chart.yaml
      name: inventory-api
      version: 0.8.1
      appVersion: "1.4.0"
      ```

    * example reason to pin chart `0.8.1`
      * chart `0.9.0` changes the security context and Deployment rollout strategy
      * staging validates that new deployment behavior first
      * production remains on `0.8.1` until the change is reviewed and promoted
      * pinning prevents production from receiving an unreviewed chart change
    * the application image can be upgraded independently when the chart exposes the image tag as a value

      ```bash
      helm upgrade --install inventory-api \
        oci://registry.example.com/charts/inventory-api \
        --version 0.8.1 \
        --values deploy/environments/production.yaml
      ```

    * a packaged chart is useful when CI, regional clusters, or other repositories must consume the same immutable deployment contract without checking out the chart source
  * optional resource example
    * `ServiceMonitor` is a custom resource supplied by Prometheus Operator
    * a cluster without the `ServiceMonitor` CRD cannot use that resource
    * the chart can render it only for clusters where Prometheus Operator is installed

      ```yaml
      # deploy/environments/production.yaml
      monitoring:
        serviceMonitor:
          enabled: true
      ```

      ```yaml
      # templates/service-monitor.yaml
      {{- if .Values.monitoring.serviceMonitor.enabled }}
      apiVersion: monitoring.coreos.com/v1
      kind: ServiceMonitor
      # ...
      {{- end }}
      ```

  * dependency example
    * the Inventory API uses Redis as a cache
    * a subchart is a Helm chart declared as a dependency of another chart
    * the Inventory API chart can declare a Redis chart as its dependency

      ```yaml
      # Chart.yaml
      dependencies:
        - name: redis
          version: 20.x.x
          repository: https://charts.example.com
          condition: redis.enabled
      ```

    * development enables the Redis subchart to install a self-contained Redis instance with the application

      ```yaml
      # deploy/environments/development.yaml
      redis:
        enabled: true
      ```

    * production disables the bundled Redis dependency and points the Inventory API at a separately managed Redis service

      ```yaml
      # deploy/environments/production.yaml
      redis:
        enabled: false

      externalRedis:
        host: inventory-cache.example.internal
      ```

  * chart maintenance cost
    * supported values are documented in `values.yaml`, the chart README, and optionally `values.schema.json`
    * chart templates and their rendered output must be reviewed and tested
    * chart and application versions represent different changes
      * increment the chart version when templates, defaults, dependencies, or the values interface changes
      * change the application version when the Inventory API release changes
    * example new option
      * production needs configurable topology spread constraints
      * the chart maintainer adds the value to `values.yaml`
      * documents and validates its structure
      * renders it in `templates/deployment.yaml`
      * tests the rendered Deployment
      * decides whether existing environment value files still render correctly
      * if the new value is optional and has a default, existing files require no change
      * if the values interface changes incompatibly, release a new major chart version
    * excessive configurability can turn `values.yaml` into a large deployment API

      ```yaml
      deployment:
        strategy: {}
        annotations: {}

      pod:
        securityContext: {}
        affinity: {}
        tolerations: []
        topologySpreadConstraints: []
        extraVolumes: []
        extraContainers: []
        extraEnv: []

      monitoring:
        serviceMonitor:
          enabled: false
      ```

    * this flexibility may be justified for a widely reused chart but is unnecessary for a small deployment with a few known variants
* choosing Helm or Kustomize
  * choose Helm when
    * deployment behavior needs a stable configuration interface
      * example
        * the application team publishes the Inventory API chart
        * customer operations teams configure image registry, database endpoint, resources, and certificates through documented values
        * consumers upgrade chart versions without editing or understanding the chart templates
      * Kustomize could represent the same deployments, but consumers would customize Kubernetes resources and patches
      * Helm is valuable here because the chart exposes a deliberate consumer-facing configuration contract
    * CI or other repositories need a versioned deployment artifact
      * example: production pins chart `0.8.1` while staging validates the deployment changes in `0.9.0`
    * optional resources belong to the supported interface
      * example: render `ServiceMonitor` only in clusters running Prometheus Operator
    * dependencies belong to the installation
      * example: bundle Redis for development but use managed Redis in production
    * Helm release operations are required
      * example: inspect release history or roll back a failed upgrade
    * one team can still benefit from Helm when it needs packaging, dependencies, hooks, tests, release history, or rollback
  * choose Kustomize when
    * environment changes are small and Kubernetes-specific
      * example: image tag, replicas, memory, namespace, and readiness timing
    * reviewers should see the exact Kubernetes fields being changed
      * a patch shows the concrete Deployment structure without tracing `.Values` through templates
    * a general consumer-facing configuration interface is not required
    * templates, loops, and conditional resources would add unnecessary indirection
    * the delivery system already owns reconciliation and configuration history
      * example: Flux builds a Kustomize overlay from Git, applies it, and corrects drift
      * Git records the desired-state history; reverting a commit restores the previous manifests for Flux to reconcile
    * concrete YAML is preferred when direct Kubernetes schema-aware editing and review are more valuable than template reuse
* using Helm and Kustomize together
  * separate workloads
    * install a vendor database from a Helm chart
    * manage the Inventory API environments with Kustomize overlays
  * organization-wide policy across vendor charts
    * Helm can pass rendered chart manifests through a Kustomize post-renderer before installation
    * the organization installs several charts it does not control
    * every workload must contain the mandatory `company.example/cost-center` label
    * the vendor charts expose different label settings, or no suitable setting
    * one centrally maintained Kustomize post-renderer adds the label consistently to the rendered resources

      ```yaml
      metadata:
        labels:
          company.example/cost-center: inventory
      ```

    * this keeps organization policy outside unrelated vendor chart templates and avoids maintaining chart forks
    * every install and upgrade must use the same post-renderer so Helm operates on repeatable rendered output
