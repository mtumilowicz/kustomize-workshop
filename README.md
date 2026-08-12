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
