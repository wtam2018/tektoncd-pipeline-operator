# Tektoncd-operator

## Dev env

### Checkout your fork

The Go tools require that you clone the repository to the
`src/github.com/openshift/tektoncd-pipeline-operator` directory in your
[`GOPATH`](https://github.com/golang/go/wiki/SettingGOPATH).

To check out this repository:

1. Create your own
   [fork of this repo](https://help.github.com/articles/fork-a-repo/)
1. Clone it to your machine:

```shell
mkdir -p ${GOPATH}/src/github.com/openshift
cd ${GOPATH}/src/github.com/openshift
git clone git@github.com:${YOUR_GITHUB_USERNAME}/tektoncd-pipeline-operator.git
cd tektoncd-pipeline-operator
git remote add upstream git@github.com:tektoncd/tektoncd-pipeline-operator.git
git remote set-url --push upstream no_push
```

### Prerequisites
You must install these tools:

1. [`go`](https://golang.org/doc/install): The language Tektoncd-pipeline-operator is
   built in
1. [`git`](https://help.github.com/articles/set-up-git/): For source control
1. [`dep`](https://github.com/golang/dep): For managing external Go
   dependencies. - Please Install dep v0.5.0 or greater.
1. [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/): For
   interacting with your kube cluster
1. operator-sdk: https://github.com/operator-framework/operator-sdk
1. minikube: https://kubernetes.io/docs/tasks/tools/install-minikube/

### Install Minikube

**create minikube instance**

```
minikube start -p mk-tekton \
 --cpus=4 --memory=8192 --kubernetes-version=v1.12.0 \
 --extra-config=apiserver.enable-admission-plugins="LimitRanger,NamespaceExists,NamespaceLifecycle,ResourceQuota,ServiceAccount,DefaultStorageClass,MutatingAdmissionWebhook"  \
 --extra-config=apiserver.service-node-port-range=80-32767
```

**set docker env**

```
eval $(minikube docker-env -p mk-tekton)
```

### Development build

1. change directory to '${GOPATH}/src/github.com/openshift/tektoncd-pipeline-operator'
```
cd ${GOPATH}/src/github.com/openshift/tektoncd-pipeline-operator
```
2. go and docker image build
``` 
operator-sdk build ${YOUR_REGISTORY}/openshift-pipelines-operator:${IMAGE_TAG}
```
3. push docker image
```
docker push ${YOUR-REGISTORY}/openshift-pipelines-operator:${IMAGE-TAG}
```
4. edit the 'image' value in deploy/operator.yaml to match to your image

### Install OLM

**clone OLM repository (into go path)**

```
git clone git@github.com:operator-framework/operator-lifecycle-manager.git \
          $GOPATH/github.com/operator-framework/
```

```
kubectl apply -f $GOPATH/github.com/operator-framework/operator-lifecycle-manager/deploy/upstream/quickstart
```

### Deploy tekton-operator

#### On minikube for testing

1. Create openshift-pipeline-operator namespace

   `kubectl create namespace openshift-pipelines-operator`

1. Apply operator crd

   `kubectl apply -f deploy/crds/*_crd.yaml`

1. Deploy the operator

    `kubectl apply -f deploy/ -n openshift-pipelines-operator`

1. Install pipeline by creating an `Install` CR

    `kubectl apply -f deploy/crds/*_cr.yaml`

### Deploy pipeline using CatalogSource on OLM

1. install minikube [see above](#install-minikube)
1. install olm [see above](#install-olm)
1. Add new catalog source **localOperators**

    `kubectl apply -f https://raw.githubusercontent.com/nikhil-thomas/operator-registry/pipeline-operator/deploy/operator-catalogsource.0.0.1.yaml`

    Once the CatalogSource has been applied, you should find it
    under `Catalog > Operator Management`  of the [web console]

1. Subscribe to `Tektoncd Operator`
    1. Open [web console]
    1. Select [`tekton-pipelines` namespace](http://localhost:9000/status/ns/tekton-pipelines)
    1. Select [`Catalog > Operator Management`](http://localhost:9000/operatormanagement/ns/tekton-pipelines)
    1. Scroll down to `Tektoncd Operator` under `localoperators`

        **NOTE:** it will take few minutes to appear after applying the `catalogsource`

    1. Click `Create Subscription` button
        1. ensure `namespace` in yaml is `tekton-pipelines` e.g.
            <details>
              <summary> sample subscription </summary>

              ```yaml
                apiVersion: operators.coreos.com/v1alpha1
                kind: Subscription
                metadata:
                  generateName: tektoncd-subscription
                  namespace: tekton-pipelines
                spec:
                  source: localoperators
                  sourceNamespace: tekton-pipelines
                  name: tektoncd
                  startingCSV: tektoncd-operator.v0.0.1
                  channel: alpha
              ```
            </details>
        1. Click `Create` button at the bottom

  1. Verify operator is installed successfully
      1. Select `Catalog > Installed operators`
      1. look for `Status` `InstallSucceeded`

1. Install Tektoncd-Pipeline by creating an `install` CR
    1. Select `Catalog > Developer Catalog`, you should find `TektonCD-Pipeline Install`
    1. Click on it and it should show the Operator Details Panel
    1. Click on `Create` which show an example as below
          <details>
              <summary> example </summary>
              ```yaml

              apiVersion: tekton.dev/v1alpha1
              kind: Install
              metadata:
                name: example
                namespace: tekton-pipelines ### must be this
              spec: {}

              ```
          </details>
    1. Verify that the pipeline is installed
        1. ensure pipeline pods are running
        1. ensure pipeline crds exist e.g. `kubectl get crds | grep tekton` should show
          ```shell
          clustertasks.tekton.dev
          installs.tekton.dev
          pipelineresources.tekton.dev
          pipelineruns.tekton.dev
          pipelines.tekton.dev
          taskruns.tekton.dev
          tasks.tekton.dev
          ```

### End to End workflow

This section explains how to test changes to the operator by executing the entire end-to-end workflow of edit, test, build, package, etc... 

It asssumes you have already followed install [minikube](#install-minikube) and [OLM](#install-olm).

#### Generate new image, CSV

1. Make changes to the operator
1. Test operator locally with `operator-sdk up local`
1. Build operator image `operator-sdk build <imagename:tag>`
1. Update image reference in `deploy/operator.yaml`
1. Build csv using `opertor-sdk olm-catalog gen-csv --csv-version 0.0.<x>` **change `<x>`**
1. Apply the CSV
    ```shell
    kubectl apply -f deploy/olm-catalog/tektoncd-operator/0.0.<x>/*.yaml
    ```
1. Verify that the new image is running

#### Update Local CatalogSource

1. clone the fork of [operator-registry fork][or-fork] where we have added tektoncd-pipeline manifests
    ```shell
    git clone https://github.com/nikhil-thomas/operator-registry
    git checkout -b pipeline-operator
    ```
2.  Copy csv from *step 5* to `manifests` directory in `operator-registry`

     **NOTE:** Be sure to preserve the directory structure
     
     **IMPORTANT:** Ensure latest crd(s) are also beside csv
     
3. Build and push operator-registry image
    ```shell
    docker build -t example-registry:latest -f upstream-example.Dockerfile
    docker push example-registry:latest
    ```
4. Update image reference in catalog-src - `deploy`
   `kubectl apply -f deploy/operator-catalogsource.0.0.1.yaml`


[web console]: http://localhost:9000
[or-fork]: https://github.com/nikhil-thomas/operator-registry/tree/pipeline-operator