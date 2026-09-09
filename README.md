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
* render
  * purpose: produce complete Kubernetes manifests without changing a cluster
  1. run Kustomize against a directory that contains a Kustomization file

     ```bash
     kubectl kustomize <kustomization-directory>
     ```

  2. Kustomize reads one Kustomization file from that directory
     * in particular: kustomization-directory = directory path, not the path to the Kustomization file
     * recognized names: `kustomization.yaml`, `kustomization.yml`, or `Kustomization`
     * more than one recognized file in the same directory causes an error
  3. Kustomize creates an in-memory resource set from the paths specified in `resources`
     * example

       ```yaml
       resources: # list of paths to manifest files or Kustomize directories
         - deployment.yaml # relative path = resolved from the Kustomization directory
       ```

     * the filename does not determine whether Kustomize uses the file as a resource
       * specifying the file path in `resources` determines this role
     * a manifest file contributes its Kubernetes objects to the current resource set
     * a Kustomize directory contributes the resource set produced by its Kustomization file
     * a YAML file that is not reachable through paths specified in `resources` is ignored
  4. Kustomize applies configured patches and transformations to the resource set
     * example
  
       ```yaml
       patches: # identifies a patch file
         - path: deployment-patch.yaml # relative path = resolved from the Kustomization directory
       ```
     * the filename does not determine whether Kustomize uses the file as a patch
       * specifying the file path in `patches[].path` determines this role
     * the patch modifies matching objects already in the resource set (already loaded by the current build)
        * in particular: the patch file is not added as a separate object
        * matching strategies
          * patch's resource identity
              * identify the target with `apiVersion`, `kind`, and `metadata.name` in the patch file
                  * example
                      ```
                      # deployment-patch.yaml
                      apiVersion: apps/v1
                      kind: Deployment
                      metadata:
                        name: inventory-api
                      ```
              * usual convention for a strategic merge patch that changes one object
                * in particular: omits explicit `target`

          * explicit `target` in `kustomization.yaml`
            * required for a JSON6902 patch
            * useful when one patch selects multiple objects or uses label, annotation, or name-pattern selection
                  * example
                    ```yaml
                    patches:
                      - path: deployment-patch.yaml
                        target: # every declared condition must match
                          group: apps
                          version: v1
                          kind: Deployment
                          name: inventory-api
                          namespace: inventory-dev
                          labelSelector: app.kubernetes.io/name=inventory-api
                    ```
          
        * a practical convention is one focused patch file per target object
          * focused means a clear target and limited scope, not one file per changed field
  5. Kustomize prints the complete resource set as Kubernetes manifests to standard output
     * inspect the output, redirect it to a file, or pass it to another command
  * notes
    * rendering does not modify source files
    * rendering does not contact a Kubernetes cluster
* apply
  * purpose: create or update Kubernetes objects from a Kustomization
  1. apply a Kustomization directory

     ```bash
     kubectl apply -k <kustomization-directory>
     ```

  2. `kubectl` renders the Kustomization
  3. `kubectl` sends the rendered objects to the Kubernetes API
  4. the Kubernetes API creates new objects and updates existing objects
* transformations
  * a transformer is Kustomize processing logic, not a Kubernetes resource
  * it changes applicable fields in the loaded resources without requiring a patch
    * not always replaceable by patch
        * example: `namespace: inventory-dev` changes both the loaded Deployment and Service
            * using patches would require separate targets for the Deployment and Service
  * types
      * changes applied across resources
        * apply the same change to every applicable loaded resource
        * do not select an individual resource by name
        * `namespace`
          * assigns a namespace to every namespace-scoped resource
          * leaves cluster-scoped resources unchanged
        * `labels`
          * adds labels to applicable resource metadata
          * can also add them to Pod templates and selectors
        * `commonAnnotations`
          * vs labels
            * labels identify and group resources
                * used by selectors and queries.
            * annotations attach non-identifying information
                * cannot be used by Kubernetes selectors
                * used by tools or humans as metadata
          * adds the same annotations to applicable resource metadata
      * changes applied to matching resources
        * select a resource or field using a configured name
        * leave non-matching resources unchanged
        * `images`
          * finds matching container image references across all loaded resources
          * changes their registry, image name, tag, or digest
          * leaves non-matching image references unchanged
        * `replicas`
          * matches a workload by `metadata.name`
          * sets its replica count
* generators
  * create new Kubernetes resources from files or values during rendering
  * `configMapGenerator`
    * creates a Kubernetes ConfigMap from files, literal values, or environment files during rendering
    * adds a content hash to the generated ConfigMap name
      * source files use the name without the hash
      * Kustomize adds the hash to the generated ConfigMap name
      * Kustomize adds the same name to references in the rendered resources
      * changing the ConfigMap content changes the hash
      * the new name changes the Pod template and triggers a rollout
  * `secretGenerator`
    * creates a Kubernetes Secret from files, literal values, or environment files during rendering
    * adds a content hash to the generated Secret name
      * source files use the name without the hash
      * Kustomize adds the hash to the generated Secret name
      * Kustomize adds the same name to references in the rendered resources
      * changing the Secret content changes the hash
      * the new name changes the Pod template and triggers a rollout
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
          ├── dev.yaml
          ├── staging.yaml
          └── prod.yaml
      ```

    * `chart/inventory-api/values.yaml` contains documented defaults
    * `environments/*.yaml` contains environment overrides
    * `values.schema.json` validates supported value types and constraints
    * `templates/` converts the values into Kubernetes resources
    * render production without installing it

      ```bash
      helm template inventory-api deploy/chart/inventory-api \
        --values deploy/environments/prod.yaml
      ```

    * install or upgrade production

      ```bash
      helm upgrade --install inventory-api deploy/chart/inventory-api \
        --namespace inventory-prod \
        --values deploy/environments/prod.yaml
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
        --values deploy/environments/prod.yaml
      ```

    * a packaged chart is useful when CI, regional clusters, or other repositories must consume the same immutable deployment contract without checking out the chart source
  * optional resource example
    * `ServiceMonitor` is a custom resource supplied by Prometheus Operator
    * a cluster without the `ServiceMonitor` CRD cannot use that resource
    * the chart can render it only for clusters where Prometheus Operator is installed

      ```yaml
      # deploy/environments/prod.yaml
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
      # deploy/environments/dev.yaml
      redis:
        enabled: true
      ```

    * production disables the bundled Redis dependency and points the Inventory API at a separately managed Redis service

      ```yaml
      # deploy/environments/prod.yaml
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
* project structure
  * base and overlay are project conventions, not Kustomize object types

    ```text
    manifests/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── overlays/
        ├── dev/
        ├── staging/
        └── prod/
    ```

  * `base/kustomization.yaml` lists the shared resource manifests

    ```yaml
    # manifests/base/kustomization.yaml
    resources:
      - deployment.yaml
      - service.yaml
    ```

  * each directory under `overlays` loads `base` and adds environment-specific configuration
  * the development overlay sets `namespace` because this workshop deploys development resources to `inventory-dev`

    ```yaml
    # manifests/overlays/dev/kustomization.yaml
    namespace: inventory-dev
    resources:
      - ../../base
    ```
