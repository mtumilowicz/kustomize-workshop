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
* transformer
  * applies a standard change across resources
  * example

    ```yaml
    # kubernetes/overlays/development/kustomization.yaml
    namespace: inventory-development # assigns the namespace
    ```

* patch
  * applies targeted changes to a resource
  * [Kubernetes recommends small patches that do one thing](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#customizing)
  * a practical convention is one overlay patch per target resource
    * focused means a clear target and limited scope, not one file per changed field
  * example

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
              env: # customizes the Deployment
                - name: LOG_LEVEL
                  value: DEBUG
    ```

  * `kubernetes/overlays/development/kustomization.yaml` registers the patch

    ```yaml
    patches:
      - path: deployment-patch.yaml
    ```