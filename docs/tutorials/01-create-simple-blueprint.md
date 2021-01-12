# Tutorial 01: Developing a simple Blueprint

This tutorial describes the basics of developing Blueprints. It covers the whole manual workflow from wrtting the Blueprint together with a Component Descriptor and storing them in a remote OCI repository.

For this tutorial, we are going to use the [NGINX ingress controller](https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx) as the example application which will get deployed via its upstream helm chart.

## Prerequisites

For this tutorial, you will need:

- the Helm (v3) commandline tool (see https://helm.sh/docs/intro/install/)
- an OCI compatible registry (e.g. GCR or Harbor)
- a Kubernetes Cluster (better use two different clusters: one which Landscaper runs in and one that NGINX gets installed into)

All example resources can be found in the folder [./resources/ingress-nginx](./resources/ingress-nginx) of this repository.

:warning: Note that the repository `eu.gcr.io/gardener-project/landscaper/tutorials` that is used throughout this tutorial is an example repository and has to be replaced with the path to your own registry if you want to upload your own artifacts.
If you do not have your own OCI registry, you can of course reuse the artifacts that we provided at `eu.gcr.io/gardener-project/landscaper/tutorials` which are publicly readable.

## Structure

- [Resources](#resources)
    - [Prepare nginx helm chart](#prepare-nginx-helm-chart)
    - [Define Component Descriptor](#define-the-component-descriptor)
    - [Create Blueprint](#create-blueprint)
    - [Render and Validate](#render-and-validate-locally)
- [Remote Upload](#remote-upload)
- [Installation](#installation)
- [Summary](#summary)
- [Up next](#up-next)

## Step 1. Prepare the NGINX helm chart

The current helm deployer only supports helm charts stored in an OCI registry. We therefore have to convert and upload the open source helm chart as an OCI artifact to our registry.

```shell script
# add open source and nginx helm registries
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add stable https://charts.helm.sh/stable
helm repo update

# download the nginx ingress helm chart and extract it to /tmp/nginx-ingress
helm pull ingress-nginx/ingress-nginx --untar --destination /tmp

# upload the helm chart to an OCI registry
export OCI_REGISTRY="eu.gcr.io" # <-- replace this with the URL of your own OCI registry
export CHART_REF="$OCI_REGISTRY/mychart/reference:my-version" # e.g. eu.gcr.io.gardener-project/landscaper/tutorials/charts/ingress-nginx:v0.1.0
export HELM_EXPERIMENTAL_OCI=1
helm registry login -u myuser $OCI_REGISTRY
helm chart save /tmp/ingress-nginx $CHART_REF
helm chart push $CHART_REF
```

## Step 2: Define the Component Descriptor

A Component Descriptor contains references and locations to all _resources_ that are used by Landscaper to deploy and install an application. In this example, the only kind of _resources_ is a `helm` chart (that of the nginx-ingress controller that we uploaded to an OCI registry in the previous step) but it could also be `oci images` or even `node modules`.

For more information about the component descriptor and the usage of the different fields, refer to the [component descriptor documentation](https://github.com/gardener/component-spec).

```yaml
meta:
  schemaVersion: v2

component:
  name: github.com/gardener/landscaper/nginx-ingress
  version: v0.2.0

  provider: internal
  sources: []
  componentReferences: []

  resources:
  - type: helm
    name: ingress-nginx-chart
    version: v0.1.0
    relation: external
    access:
      type: ociRegistry
      imageReference: eu.gcr.io.gardener-project/landscaper/tutorials/charts/ingress-nginx:v0.1.0
```

## Step 3: Create a Blueprint

Blueprints describe how _DeployItems_ are created by taking the values of `imports` and applying them to templates inside `deployExecutions`. Additionally, they specify which pieces of data appear as `exports` from the executed _DeployItems_.

For detailed documentation about Blueprints, look at [docs/usage/Blueprints.md](/docs/usage/Blueprints.md).

### Imports and Exports declaration

The `imports` are described as a list of import declarations. Each _import_ is declared by a unique name and a type which is either a JSON schema or a `targetType`.

<details>

Imports with the type `schema` import their data from a data object with a given JSON schema. Imports with the type `targetType` are imported from the specified _Target_.

```yaml
# import with type JSON schema

- name: myimport
  schema: # valid jsonschema
    type: string | object | number | ...
```

```yaml
# import from/into targetType

- name: myimport
  targetType: "" # e.g. landscaper.gardener.cloud/kubernetes-cluster
```

</details>

Our _nginx-ingress_ cotroller in this tutorial only needs to import a target Kubernetes cluster. It will be used as the target cluster the Helm chart gets deployed to. Therefore, the following YAML snippet _declares_ the import:

```yaml
imports:
- name: cluster
  targetType: landscaper.gardener.cloud/kubernetes-cluster
```

The declaration of `exports` works just like declaring `imports`. Again, each _export_ is declared as a list-item with a unique name and a data type (again, JSON schema or `targetType`).

To be able to use the ingress in a later Blueprint or Installation, this Blueprint will export the name of the ingress class as a simple string. With this piece of YAML, the export is _declared_:

```yaml
exports:
- name: ingressClass
  schema: # here comes a valid jsonschema
    type: string
```

### DeployItems

_DeployItems_ are created from templates as given in the `deployExecutions` section. Each element specifies a templating step which results in one or multiple _DeployItems_ (returned in  a list).

```yaml
- name: "unique name of the deployitem"
  type: landscaper.gardener.cloud/helm | landscaper.gardener.cloud/container | ... # deployer identifier
  # names of other deployitems that the deploy item depends on.
  # If a item depends on another, the landscaper ensures that dependencies are executed and reconciled before the item.
  dependsOn: []
  config: # provider specific configuration
    apiVersion: mydeployer.landscaper.gardener.cloud/test
    kind: ProviderConfiguration
    ...
```

Currently [GoTemplate](https://golang.org/pkg/text/template/) and [Spiff](https://github.com/mandelsoft/spiff) are supported templating engines. For detailed information about the template executors, [read this](/docs/usage/TemplateExecutors.md).

While processing the templates, Landscaper offers access to the `imports` and the fields of the component descriptor through the following structure:

```yaml
imports:
  <import name>: <data value> or <target custom resource>
cd:
 component:
   resources: ...
```

Exports can be described the same way as imports.
Also exports can be templated using templating in the `exportExecutions`.
The export execution are expected to output the exports as a map of <export name>: <value> .<br>
If a target is exported the following structure is expected to be exported:
```yaml
<target export name>:
  annotations: {} # optional
  lables: {} # optional
  config:
    type: ""
    config: {}
```

In order to export values of deploy items and installations, the landscaper give access to these values via templating imports:
```yaml
values:
  deployitems:
    <deploy item name>: <deploy item export value (is type specific)>
  dataobjects:
    <data object name>: <data of the dataobject> (currently only exports of the subinstallations are accessible)
  targets:
    <target name>: <the target cr>
```

```yaml
apiVersion: landscaper.gardener.cloud/v1alpha1
kind: Blueprint

imports:
- name: cluster
  targetType: landscaper.gardener.cloud/kubernetes-cluster

deployExecutions:
- name: default
  type: GoTemplate
  template: |
    deployItems:
    - name: deploy
      type: landscaper.gardener.cloud/helm
      target: 
        name: {{ .imports.cluster.metadata.name }}
        namespace: {{ .imports.cluster.metadata.namespace }}
      config:
        apiVersion: helm.deployer.landscaper.gardener.cloud/v1alpha1
        kind: ProviderConfiguration
        
        chart:
          {{ $resource := getResource .cd "name" "ingress-nginx-chart" }}
          ref: {{ $resource.access.imageReference" }}
        
        updateStrategy: patch
        
        name: test
        namespace: default
        
        exportsFromManifests:
        - key: ingressClass
          jsonPath: .Values.controller.ingressClass

exportExecutions:
- name: default
  type: GoTemplate
  template: |
    exports:
      ingressClass: {{ .values.deployitems.deploy.ingressClass }}

exports:
- name: ingressClass
  schema: # here comes a valid jsonschema
    type: string
```

A blueprint is defined by a directory that contains the above described Blueprint Manifest as file called `blueprint.yaml`.
The directory can contain any other data that is necessary for the deployment/templating.
For an example see [./resources/ingress-nginx/blueprint](resources/ingress-nginx/blueprint).

##### Render and Validate locally

The blueprint will result in a deploy item of helm that is templated using one import.
This resulting deploy item can be rendered and the templating can be tested by 
1. providing some sample import as file (e.g. docs/tutorials/resources/ingress-nginx/import-values.yaml)
   ```
   imports:
     cluster:
       metadata:
         name: my-target
         namespace: test
       spec:
         type: ""
         config:
           kubeconfig: |
             apiVersion: ...
   ```
2. and render the blueprint with the component descriptor: 
   ```
   landscaper-cli blueprints render ./docs/tutorials/resources/ingress-nginx/blueprint \
      -c ./docs/tutorials/resources/ingress-nginx/component-descriptor.yaml \
      -f ./docs/tutorials/resources/ingress-nginx/import-values.yaml
   ```
   
   ```
   --------------------------------------
   -- deployitems deploy
   --------------------------------------
   apiVersion: landscaper.gardener.cloud/v1alpha1
   kind: DeployItem
   metadata:
     annotations:
       execution.landscaper.gardener.cloud/dependsOn: ""
       landscaper.gardener.cloud/operation: reconcile
     creationTimestamp: null
     labels:
       execution.landscaper.gardener.cloud/name: deploy
   spec:
     config:
       apiVersion: helm.deployer.landscaper.gardener.cloud/v1alpha1
       chart:
         ref: eu.gcr.io/myproject/charts/nginx-ingress:v0.1.0
       exportsFromManifests:
       - jsonPath: .Values.controller.ingressClass
         key: ingressClass
       kind: ProviderConfiguration
       name: test
       namespace: default
       updateStrategy: patch
     target:
       name: my-cluster
       namespace: <no value>
     type: landscaper.gardener.cloud/helm
   status:
     observedGeneration: 0
   ```


### Remote Upload

After the blueprint is build it has to be uploaded to the oci registry and the reference needs to be added to the component descriptor.
The blueprint can be easily uploaded by using the landscaper cli tool which packages the blueprint and uploads to the given oci registry.

To install the landscaper see [Landscaper CLI Installation](https://github.com/gardener/landscapercli/blob/master/docs/installation.md)

```shell script
# landscaper-cli blueprints push myregistry/mypath/ingress-nginx:v0.1.0 docs/tutorials/resources/ingress-nginx/blueprint
landscaper-cli blueprints push eu.gcr.io/gardener-project/landscaper/tutorials/blueprints/ingress-nginx:v0.1.0 docs/tutorials/resources/ingress-nginx/blueprint
```

Blueprints are also just resources/artifacts of a component descriptor.
Therefore, after the blueprint is uploaded, the reference to that blueprint has to be added to the component descriptor.
This is done to ensure that all resources of a application are known and stored.
In addition, it is used by the landscaper to resolve the location of the blueprint resource.

Note that the repository context as well as the blueprint resource should be added to the component descriptor
```yaml
meta:
  schemaVersion: v2

component:
  name: github.com/gardener/landscaper/ingress-nginx
  version: v0.2.0

  provider: internal
  sources: []
  componentReferences: []

  respositoryContext:
  - type: ociRegistry
    baseUrl: eu.gcr.io/gardener-project/landscaper/tutorials/components

    
  resources:
  - type: blueprint
    name: ingress-nginx-blueprint
    relation: local
    access:
      type: ociRegistry
      imageReference: eu.gcr.io/gardener-project/landscaper/tutorials/blueprints/ingress-nginx:v0.2.0
  - type: helm
    name: ingress-nginx-chart
    version: v0.1.0
    relation: external
    access:
      type: ociRegistry
      imageReference: eu.gcr.io/gardener-project/landscaper/tutorials/charts/ingress-nginx:v0.1.0
```

Then the component descriptor can be uploaded to a oci registry using again the landscaper cli.
When the upload succeeds, the component should be accessible at `eu.gcr.io/my-project/comp/github.com/gardener/landscaper/ingress-nginx/v0.1.0` in the registry.
```shell script
landscaper-cli cd push docs/tutorials/resources/ingress-nginx/component-descriptor.yaml
```

### Installation

As all external resources are defined and uploaded, the nginx ingress can be installed into the second kubernetes cluster.

Before the runtime resources are defined, the landscaper controller has to be installed into the first kubernetes cluster.
For a detailed installation instructions see [Landscaper Controller Installation](../gettingstarted/install-landscaper-controller.md).

The previously created blueprint can be installed in a target system by instructing the landscaper via a Installation resource to install it to the second cluster.

The blueprint defines one import of a kubernetes, therefore, a `Target` resource of type `landscaper.gardener.cloud/kubernetes-cluster` that points to the second cluster has to be defined.

See the example of a `Target` below.
```yaml
apiVersion: landscaper.gardener.cloud/v1alpha1
kind: Target
metadata:
  name: my-cluster
spec:
  type: landscaper.gardener.cloud/kubernetes-cluster
  config:
    kubeconfig: |
      apiVersion:...
      # here goes the kubeconfig of the target cluster
```

An Installation is an instance of a blueprint, which means that it is the runtime representation of one specific blueprint installation.

The installation consists of a blueprint, imports and exports.<br>
__blueprint__:
To reference the previously uploaded Blueprint, the component descriptor is referenced by specifying the repository context, the component name and version.
With that, the landscaper is able to resolve the component descriptor from the oci registry.<br>
The Blueprint artifact in the component descriptor is specified as resources with the unique name `ingress-nginx-blueprint` 
which can be referenced with `resourceName: ingress-nginx-blueprint`
```yaml
spec:
  blueprint:
    ref:
      repositoryContext:
        type: ociRegistry
        baseUrl: eu.gcr.io/gardener-project/landscaper/tutorials/components
      componentName: github.com/gardener/landscaper/ingress-nginx
      version: v0.2.0
```

__imports__:
The blueprint needs a target import of type kubernetes cluster.
The target `mycluster` is created as mentioned above and has to be connected to the import.
This is done by specifying the target as targets import.

:warning: The "#" has to be used to reference the previously created target. Otherwise, the landscaper would try to import the target from another component's export.
```yaml
imports:
  targets:
  - name: cluster
    # the "#" forces the landscaper to use the target with the name "my-cluster" in the same namespace
    target: "#my-cluster"
```

__exports__:

The nginx ingress blueprint export the used ingressClass so that it can be reused by other components.
To give the generic ingress class more semantic meaning in the current installation, the export is exported as `myIngressClass`.
Other installation are now able to consume the data with this specific name.

:warning: Note that this name has to be unique so that it will not be overwritten by other installations.

The export is a data object export, therefore the eyport is defined under `spec.exports.data` and is written to the `dataRef: myIngressClass`.
```yaml
exports:
  data:
  - name: ingressClass
    dataRef: "myIngressClass"
```

```yaml
apiVersion: landscaper.gardener.cloud/v1alpha1
kind: Installation
metadata:
  name: my-ingress
spec:
  blueprint:
    ref:
      repositoryContext:
        type: ociRegistry
        baseUrl: eu.gcr.io/gardener-project/landscaper/tutorials/components
      componentName: github.com/gardener/landscaper/ingress-nginx
      version: v0.2.0
      resourceName: ingress-nginx-blueprint

  imports:
    targets:
    - name: cluster
      # the "#" forces the landscaper to use the target with the name "my-cluster" in the same namespace
      target: "#my-cluster"
  
  exports:
    data:
    - name: ingressClass
      dataRef: "myIngressClass"
```

When the `Target` and the Installation CRs are properly configured, they can be applied to the kubernetes cluster running the landscaper.

```shell script
kubectl create -f docs/tutorials/resources/ingress-nginx/my-target.yaml
kubectl create -f docs/tutorials/resources/ingress-nginx/installation.yaml
```

The landscaper will then immediately start to reconcile the installation as all imports are satisfied.

The first resources that will be created is the execution object which is a helper resource that contains the rendered deployitems.
The status shows the one specified Helm deploy item which has been automatically created by the landscaper.
```shell script
$ kubectl get inst
NAME                        PHASE       CONFIGGEN   EXECUTION                   AGE
my-ingress                  Succeeded               my-ingress                  4m11s

$ kubectl get exec my-execution -oyaml
apiVersion: landscaper.gardener.cloud/v1alpha1
kind: Execution
metadata:
  ...
spec:
  deployItems:
  - config:
      apiVersion: helm.deployer.landscaper.gardener.cloud/v1alpha1
      chart:
        ref: eu.gcr.io/gardener-project/landscaper/tutorials/charts/ingress-nginx:v0.1.0
      exportsFromManifests:
      - jsonPath: .Values.controller.ingressClass
        key: ingressClass
      kind: ProviderConfiguration
      name: test
      namespace: default
    name: deploy
    target:
      name: ts-test-cluster
      namespace: default
    type: landscaper.gardener.cloud/helm
status:
status:
  ...
  deployItemRefs:
  - name: deploy
    ref:
      name: my-ingress-deploy-xxx
      namespace: default
      observedGeneration: 1
  ...
```

The newly created deploy item will be reconciled by the Helm deployer.
The helm deployer actually creates and updates the configured resources of the helm chart in the target cluster.
When the deployer successfully reconciles the deploy item, the phase is set to `Succeeded` and the all managed resources are added to the DeployItem's status.

```shell script
$ kubectl get di my-ingress-deploy-xxx -oyaml
apiVersion: landscaper.gardener.cloud/v1alpha1
kind: DeployItem
metadata:
  ...
spec:
  config:
    apiVersion: helm.deployer.landscaper.gardener.cloud/v1alpha1
    chart:
      ref: eu.gcr.io/gardener-project/landscaper/tutorials/charts/ingress-nginx:v0.1.0
    exportsFromManifests:
    - jsonPath: .Values.controller.ingressClass
      key: ingressClass
    kind: ProviderConfiguration
    name: test
    namespace: default
  target:
    name: ts-test-cluster
    namespace: default
  type: landscaper.gardener.cloud/helm
status:
  exportRef:
    name: my-ingress-deploy-5stgr-export
    namespace: default
  phase: Succeeded
  providerStatus:
    apiVersion: helm.deployer.landscaper.gardener.cloud/v1alpha1
    kind: ProviderStatus
    managedResources:
    - apiVersion: rbac.authorization.k8s.io/v1
      kind: Role
      name: test-ingress-nginx
      namespace: default
    ...
```

The blueprint also configures export values.
Therefore, the helm deployer also creates a secret that contains the exported values.

```shell script
# A kubectl plugin is used to automatically decode the base64 encoded secret
$ kubectl ksd get secret my-ingress-deploy-5stgr-export -oyaml
apiVersion: v1
kind: Secret
metadata:
  ...
stringData:
  config: |
    ingressClass: nginx
type: Opaque
```

This exported value is then propagated to the execution object and then used in the `exportExecutions` to create the exports.
The execution resource combines all deployitem exports into a data object.
```shell script
$ kubectl get exec my-execution
NAME                        PHASE       EXPORTREF                          AGE
my-ingress                  Succeeded   3a4cwhagjhl5i6iu3vvljkjkzffxbk4p   5m

$ kubectl get do 3a4cwhagjhl5i6iu3vvljkjkzffxbk4p -oyaml
apiVersion: landscaper.gardener.cloud/v1alpha1
kind: DataObject
metadata:
  ...
data:
  deploy:
    ingressClass: nginx
```

The landscaper collects the export from the execution and creates the configured exported dataobject `myIngressClass`.
The exported dataobject is a contextified dataobject, which means that it can only be imported by other installations in the same context.
The dataobject's context is the root context `""` so that all root installations could use the export as import.

Contextified dataobjects name is a hash of the exported key and the context, so that they can be unqiely identified by the landscaper.
:warning: Note: also targets are contextified but global target/dataobjects can be referenced with a prefix `#` as in the current target import.

```shell script
$ kubectl get do -l data.landscaper.gardener.cloud/key=myIngressClass
NAME                               CONTEXT   KEY
dole6tby5kerlxruq2n2efxiql6onp3h             myIngressClass

$ kubectl get do -l data.landscaper.gardener.cloud/key=myIngressClass -oyaml
apiVersion: v1
kind: List
items:
- apiVersion: landscaper.gardener.cloud/v1alpha1
  kind: DataObject
  metadata:
    creationTimestamp: "2020-10-12T10:11:01Z"
    generation: 1
    labels:
      data.landscaper.gardener.cloud/key: myIngressClass
      data.landscaper.gardener.cloud/source: Installation.default.my-ingress
      data.landscaper.gardener.cloud/sourceType: export
    name: dole6tby5kerlxruq2n2efxiql6onp3h
    namespace: default
  data: nginx
```

### Summary
- A blueprint has been created that describes how a nginx ingress can be deployed into a kubernetes cluster.
- A component descriptor has been created that contains the blueprint and another external resources as resources.
- The blueprint and the component descriptor are uploaded to the oci registry.
- A installation has been defined and applied to the cluster which resulted in teh deployed nginx application. 

### Up Next
In the [next tutorial](./02-simple-import.md), another application is deployed that used the exported ingressClass data.
