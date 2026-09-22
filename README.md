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
  * customizes Kubernetes manifests without introducing a template language
  * uses the same Kubernetes manifests as input to multiple builds
    * example: development and production builds reference the same input manifests
  * lets each build apply its own transformations and patches
    * changes affect only the rendered output of that build
    * input manifests remain unchanged
    * changes in one build do not affect other builds
  * avoids maintaining a separate copy of the complete manifests for each environment
  * makes environment-specific changes visible as transformations and patches
  * emits standard Kubernetes YAML
  * example
    * requirement: add the same readiness probe to every environment
    * without Kustomize
      * repeat this block in three Deployment files

        ```yaml
        # base/deployment.yaml (partial snippet)
        readinessProbe:
          httpGet:
            path: /ready
            port: http
        ```

    * with Kustomize
      * add the block once to `base/deployment.yaml`
      * development, staging, and production inherit it
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
     * Kustomize creates an in-memory resource set from the paths specified in `resources`
       * example

         ```yaml
         # kustomization.yaml (partial snippet)
         resources: # list of paths to manifest files or Kustomize directories
           - deployment.yaml # relative path = resolved from the Kustomization directory
         ```

       * the filename does not determine whether Kustomize uses the file as a resource
         * specifying the file path in `resources` determines this role
       * a manifest file contributes its Kubernetes objects to the current resource set
       * a Kustomize directory contributes the resource set produced by its Kustomization file
       * a YAML file that is not reachable through paths specified in `resources` is ignored
  3. Kustomize runs generators and adds the generated objects to the resource set
     * generator types
       * `configMapGenerator`
         * creates a Kubernetes `ConfigMap`
         * stores non-sensitive configuration as readable data
         * accepts files, literal values, or environment files
       * `secretGenerator`
         * creates a Kubernetes `Secret`
         * stores sensitive values as base64-encoded data
           * base64 encoding is not encryption
         * accepts files, literal values, or environment files
         * creates an `Opaque` Secret by default
           * another Secret type can be specified
       * `helmCharts`
         * uses a Helm chart to generate Kubernetes objects
         * requires the Helm command to be installed and available on `PATH`
         * requires the `--enable-helm` option
         * example: render a remote Helm chart with overridden values

           ```yaml
           # kustomization.yaml (partial snippet)
           helmCharts:
             - name: minecraft
               repo: https://itzg.github.io/minecraft-server-charts
               version: 3.1.3
               releaseName: moria
               valuesInline:
                 minecraftServer:
                   eula: true
                   difficulty: hard
           ```

           * Kustomize starts the Helm command
           * Helm downloads chart version `3.1.3` from the specified repository
           * Kustomize passes `releaseName` and `valuesInline` to `helm template`
           * Helm returns the generated Kubernetes manifests to Kustomize
             * Helm does not install them
           * Kustomize adds each Kubernetes object declared in those manifests to the resource set
             * example: a Deployment, Service, or ConfigMap
       * `generators`
         * runs custom generator plugins specified by file
         * requires the `--enable-alpha-plugins` option
         * example: decrypt a SOPS-encrypted Kubernetes Secret with the KSOPS plugin

           ```yaml
           # kustomization.yaml (partial snippet)
           generators:
             - secret-generator.yaml
           ```

           ```yaml
           # secret-generator.yaml
           apiVersion: viaduct.ai/v1
           kind: ksops
           metadata:
             name: secret-generator
           files:
             - database-secret.enc.yaml
           ```

           ```yaml
           # database-secret.enc.yaml (partial snippet)
           apiVersion: v1
           kind: Secret
           metadata:
             name: database
           type: Opaque
           stringData:
             password: ENC[AES256_GCM,data:...,iv:...,tag:...,type:str]
           sops:
             age:
               - recipient: age1example...
                 enc: |
                   -----BEGIN AGE ENCRYPTED FILE-----
                   ...
                   -----END AGE ENCRYPTED FILE-----
             # Additional SOPS metadata is omitted from this example.
           ```

           * `database-secret.enc.yaml` is a Kubernetes Secret manifest
           * SOPS encrypts the value of `stringData.password`
           * `sops.age[].recipient` identifies the public key used to encrypt the Secret
           * the matching private key is stored outside the repository
           * `SOPS_AGE_KEY_FILE` specifies the private-key file used during rendering
             * example: `SOPS_AGE_KEY_FILE=/secure/keys/age-keys.txt`
           * Kustomize runs the installed KSOPS plugin
           * KSOPS uses the matching private key to decrypt the encrypted values
           * KSOPS returns the Kubernetes Secret manifest with its decrypted values
           * Kustomize adds that Kubernetes Secret to the resource set
           * requires the KSOPS plugin and the `--enable-alpha-plugins` and `--enable-exec` options
     * generated ConfigMaps and Secrets
       * have a content hash added to their names
         * source configuration uses the name without the hash
         * Kustomize updates references to use the generated name
         * changing the content changes the hash
         * the changed reference updates the Pod template and triggers a rollout
  4. Kustomize transforms the resource set
     * built-in transformations
       * a transformer is Kustomize processing logic, not a Kubernetes resource
       * transformations are configured in `kustomization.yaml`
       * changes applied across loaded resources
         * `namespace`
           * sets `metadata.namespace` on every namespace-scoped object in loaded resources
           * example: `namespace: inventory-dev` produces `metadata.namespace: inventory-dev`
           * leaves cluster-scoped objects unchanged
           * practice
             * omit `metadata.namespace` from reusable base manifests
             * set `namespace` in each overlay

               ```yaml
               # overlays/dev/kustomization.yaml
               namespace: app-dev
               resources:
                 - ../../base
               ```

             * another overlay can load the same base and set a different namespace
             * `metadata.namespace` can also be changed with a patch, but the `namespace` transformer is simpler and standard
         * `namePrefix`
           * adds a prefix to the name of every object in loaded resources
           * updates recognized fields that reference those object names
           * example

             ```yaml
             # resources (partial snippet)
             apiVersion: v1
             kind: ServiceAccount
             metadata:
               name: application
             ---
             apiVersion: apps/v1
             kind: Deployment
             spec:
               template:
                 spec:
                   serviceAccountName: application
             ```

             ```yaml
             # kustomization.yaml (partial snippet)
             namePrefix: dev-
             ```

             * the ServiceAccount name becomes `dev-application`
             * `serviceAccountName` also becomes `dev-application`
           * other recognized references include
             * `configMapRef.name`
             * `secretRef.name`
             * `volumes[].configMap.name`
             * `volumes[].secret.secretName`
           * Kustomize does not update fields that it does not recognize as name-reference fields
             * use `replacements` to copy the transformed object name into those fields
         * `nameSuffix`
           * adds a suffix to the name of every object in loaded resources
           * updates recognized fields that reference those object names
         * `labels`
           * adds labels to `metadata.labels` of every object in loaded resources
           * can also add labels to Pod templates and selectors
         * `commonAnnotations`
           * adds annotations to `metadata.annotations` of every object in loaded resources
           * labels and annotations are both Kubernetes metadata
             * labels identify or group objects
             * Kubernetes selectors can use labels
             * annotations store descriptive or tool-specific information
             * Kubernetes selectors cannot use annotations
       * changes applied to matching values or objects in loaded resources
         * `images`
           * finds matching container image references in loaded resources
           * `name` matches the image name in `containers[].image`
           * changes the registry, image name, tag, or digest
           * use case: use a different image version in each environment
               * example
    
                 ```yaml
                 # deployment.yaml (partial snippet)
                 containers:
                   - name: api
                     image: example-app:1.0
                 ```
    
                 ```yaml
                 # overlays/dev/kustomization.yaml (partial snippet)
                 images:
                   - name: example-app
                     newTag: 1.1-rc
                 ```
    
                 ```yaml
                 # overlays/prod/kustomization.yaml (partial snippet)
                 images:
                   - name: example-app
                     newTag: 1.1
                 ```
    
                 * the development image becomes `example-app:1.1-rc`
                 * the production image becomes `example-app:1.1`
         * `replicas`
           * finds workloads with matching `metadata.name` in loaded resources
           * sets `spec.replicas`
           * supports Deployments, StatefulSets, ReplicaSets, and ReplicationControllers
           * use case: use a different replica count in each environment
               * example
    
                 ```yaml
                 # deployment.yaml (partial snippet)
                 metadata:
                   name: api-deployment
                 spec:
                   replicas: 2
                 ```
    
                 ```yaml
                 # overlays/dev/kustomization.yaml (partial snippet)
                 replicas:
                   - name: api-deployment
                     count: 1
                 ```
    
                 ```yaml
                 # overlays/prod/kustomization.yaml (partial snippet)
                 replicas:
                   - name: api-deployment
                     count: 3
                 ```
    
                 * the development Deployment has `spec.replicas: 1`
                 * the production Deployment has `spec.replicas: 3`
         * `replacements`
           * reads a field from one object in loaded resources
           * copies the value to selected fields in other loaded resources
           * use case: copy a transformed resource name into a field that Kustomize does not recognize as a name reference
             * example: copy the renamed Service name into a container environment variable

                 ```yaml
                 # service.yaml (partial snippet)
                 kind: Service
                 metadata:
                   name: backend
                 ```
    
                 ```yaml
                 # deployment.yaml (partial snippet)
                 kind: Deployment
                 metadata:
                   name: api-deployment
                 spec:
                   template:
                     spec:
                       containers:
                         - name: api
                           env:
                             - name: BACKEND_SERVICE
                               value: backend
                 ```
    
                 ```yaml
                 # kustomization.yaml (partial snippet)
                 namePrefix: dev-
                 replacements:
                   - source:
                       kind: Service
                       name: backend
                       fieldPath: metadata.name
                     targets:
                       - select:
                           kind: Deployment
                           name: api-deployment
                         fieldPaths:
                           - spec.template.spec.containers.[name=api].env.[name=BACKEND_SERVICE].value
                 ```
    
                 * `namePrefix` changes the Service name from `backend` to `dev-backend`
                 * `replacements` copies `dev-backend` to the `BACKEND_SERVICE` value
     * patches
       * example

         ```yaml
         # kustomization.yaml (partial snippet)
         patches: # identifies a patch file
           - path: deployment-patch.yaml # relative path = resolved from the Kustomization directory
         ```

       * the filename does not determine whether Kustomize uses the file as a patch
         * specifying the file path in `patches[].path` determines this role
       * the patch modifies matching objects already in the resource set (already loaded by the current build)
         * in particular: the patch file is not added as a separate object
         * patch types
           * strategic merge patch
             * is written as a Kubernetes object with `apiVersion`, `kind`, `metadata`, and the fields to modify
             * contains
               * fields that identify the target object
               * fields that identify nested list items
                 * example: `containers[].name` identifies the container to modify

                   ```yaml
                   # deployment-patch.yaml (partial snippet)
                   containers:
                     - name: inventory-api
                       imagePullPolicy: Always # Kustomize changes imagePullPolicy.
                   ```

               * fields to add, change, or delete
                 * example: add `team=platform` to `metadata.labels` of the target object

                   ```yaml
                   # deployment-patch.yaml (partial snippet)
                   metadata:
                     labels:
                       team: platform
                   ```

             * unchanged fields can be omitted
             * targeting by resource identity
               * when `target` is omitted, Kustomize compares `apiVersion`, `kind`, and `metadata.name`
               * Kustomize applies the patch to the object with matching identity
               * example

                 ```yaml
                 # kustomization.yaml (partial snippet)
                 patches:
                   - path: replicas-patch.yaml
                 ```

                 ```yaml
                 # replicas-patch.yaml
                 apiVersion: apps/v1
                 kind: Deployment
                 metadata:
                   name: inventory-api
                 spec:
                   replicas: 3
                 ```

             * targeting with `target`
               * a strategic merge patch can include `target`
                 * when `target` is present
                   * only fields specified in `target` select the objects
                   * `metadata.name` in the patch is required but does not participate in selection
                   * `apiVersion` and `kind` still define how Kustomize interprets the patch
               * `target` can contain `group`, `version`, `kind`, `name`, `namespace`, `labelSelector`, and `annotationSelector`
               * every field specified in `target` must match
               * use case: apply the same change to a group of objects
                 * example: add `team=platform` to every Deployment labeled `env=dev`

                   ```yaml
                   # kustomization.yaml (partial snippet)
                   patches:
                     - path: team-patch.yaml
                       target:
                         kind: Deployment
                         labelSelector: env=dev
                   ```

                   ```yaml
                   # team-patch.yaml
                   apiVersion: apps/v1
                   kind: Deployment
                   metadata:
                     name: required-placeholder
                     labels:
                       team: platform
                   ```

           * JSON Patch
             * describes field changes as operations
               * supported operations include `add`, `remove`, and `replace`
             * contains no Kubernetes resource identity
             * requires `target` in the Kustomization
             * example

               ```yaml
               # kustomization.yaml (partial snippet)
               patches:
                 - path: replicas-patch.yaml
                   target:
                     group: apps
                     version: v1
                     kind: Deployment
                     name: inventory-api
               ```

               ```yaml
               # replicas-patch.yaml
               - op: replace
                 path: /spec/replicas
                 value: 3
               ```
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

## Kustomize and Helm

* purpose
  * Kustomize customizes Kubernetes resources for different targets
  * Helm packages, distributes, installs, and upgrades Kubernetes applications as charts and releases
  * Helm can package both third-party and company-owned applications
    * a company-owned application can provide its deployment as a maintained product
      * the application team owns and versions the chart
      * documented values define what consumers can configure
      * chart releases define changes to deployment behavior
      * consumers select a chart version and values instead of editing or copying Kubernetes manifests
* source format
  * Kustomize starts from Kubernetes resource YAML and applies transformations and patches
  * Helm charts use Go templates and values to generate Kubernetes resource YAML
* configuration
  * Kustomize expresses variants with bases and overlays
  * Helm exposes chart-defined settings through values
* lifecycle
  * `kubectl kustomize` renders manifests and does not track releases
  * Helm tracks each installed chart instance as a release and supports upgrades and rollbacks
* choosing Helm or Kustomize
  * choose Helm when
    * other teams need to deploy the application through documented values
      * example
        * the application team publishes the Inventory API chart
        * customer operations teams configure image registry, database endpoint, resources, and certificates through documented values
        * consumers upgrade chart versions without editing the chart templates
      * with Kustomize, consumers would need to work directly with Kubernetes manifests and patches
    * the chart controls which optional resources are installed
      * example: render `ServiceMonitor` only in clusters running Prometheus Operator
    * dependent applications can be part of the installation
      * example: bundle Redis for development but use managed Redis in production
    * Helm release operations are required
      * example: inspect release history or roll back a failed upgrade
    * one team can still benefit from Helm when it needs packaging, dependencies, hooks, tests, release history, or rollback
  * choose Kustomize when
    * users are expected to work directly with Kubernetes manifests and fields
    * deployment variants only change fields in existing Kubernetes objects
      * example: image, replica count, namespace, or memory limit
    * the application does not need to be distributed as a versioned chart
    * another system manages deployment and reconciliation
      * example: Flux
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
      # rendered resource (partial snippet)
      metadata:
        labels:
          company.example/cost-center: inventory
      ```

    * this keeps organization policy outside unrelated vendor chart templates and avoids maintaining chart forks
    * every install and upgrade must use the same post-renderer so Helm operates on repeatable rendered output
## Project structure

* example

  ```text
  manifests/ // `base` and `overlay` are project conventions, not Kustomize object types
  ├── base/
  │   ├── deployment.yaml
  │   ├── service.yaml
  │   └── kustomization.yaml
  └── overlays/
      ├── dev/
      │   └── kustomization.yaml
      ├── staging/
      │   └── kustomization.yaml
      └── prod/
          └── kustomization.yaml
  ```

* the base Kustomization loads the Deployment and Service manifests

  ```yaml
  # manifests/base/kustomization.yaml
  resources:
    - deployment.yaml
    - service.yaml
  ```

* each overlay loads the base and specifies changes for one environment
  * example: the development overlay sets the namespace and replica count

    ```yaml
    # manifests/overlays/dev/kustomization.yaml
    resources:
      - ../../base
    namespace: inventory-dev
    replicas:
      - name: inventory-api
        count: 1
    ```
