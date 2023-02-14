# Tekton/ Tekton Catalog

Tekton catalog includes a series of tasks, pipelines and resources that can be reusable across workflows.

The catalog, at public level is available at https://github.com/tektoncd/catalog

Tekton Hub offers many types of tasks and pipelines that are available to install in clusters. Available at https://hub.tekton.dev/

Tasks can be Cluster scoped and Namespaces.
The only difference, is their scope. While clusterTasks can be used by all users in the cluster (usually when the Tekton Operator installs) these are available to all users, while Tasks are only available for a particular namespace.

- To install a task:
  - K apply -f build.yaml
  
- See installed tasks
  - K get tasks


Even though tasks can be triggered by the use of web hooks, which trigger pipeline runs through trigger bindings and templates, these can also be manually run by providing the values using a manifest file that interacts with the CRD:

``` bash

apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: example-run
spec:
  taskRef:
    name: golang-build
  params:
  - name: package
    value: github.com/tektoncd/pipeline
  workspaces:
  - name: source
    persistentVolumeClaim:
      claimName: my-source

```

- Create the Task Run:
  - K apply -f taskrun.yaml
- Get the status of the Task Ru:
- K get taskrun example-run -oyaml


## Tekton Catalog at Ford

To start our integration we part from a Dockerfile:

``` docker

FROM alpine

ENTRYPOINT echo hello medium

```

Multiple teams have catalogs with shared tasks:

- https://hub.cicd.ford.com/ - Ford’s TektonHUB	

FADS Team: https://github.ford.com/FADS/tekton-catalog

## Tekton Quickstart, using Tekton catalog

Pre-req

- A kubernetes Cluster (I used minukube locally)
- Tekton ClI/Tekton pipelines
- Image registry account, for authentication
- Repository

### Step 1 - Install Tekton CLI / Dashboard

``` bash

brew install tektoncd-cli \
kubectl apply -f https://storage.googleapis.com/tekton-releases/operator/latest/release.yaml &&
kubectl apply -f https://raw.githubusercontent.com/tektoncd/operator/main/config/crs/kubernetes/config/all/operator_v1alpha1_config_cr.yaml 


```

You can access the tekton dashboard using port forward on the dashboard service:

``` bash

kubectl port-forward svc/tekton-dashboard 9097:9097 -n tekton-pipelines

```

### Step 2 - Automate our CI/CD

Now we need tasks to run the build manual steps of our image and will use tekton hub to pull the clone task from it to start our process:

#### Git Clone task

``` bash

tkn hub install task git-clone

```

#### Push/Pull task

We will use buildah task to build/push our app:

``` yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: buildah
spec:
  params:
    - name: IMAGE
      description: Reference of the image buildah will produce.
    - name: STORAGE_DRIVER
      description: Set buildah storage driver
      default: overlay
    - name: DOCKERFILE
      description: Path to the Dockerfile to build.
      default: ./Dockerfile
    - name: IMAGE_PUSH_SECRET_NAME
      description: Kubernetes secrets contain image push username and password
  workspaces:
    - name: source
  steps:
    - name: build
      image: quay.io/buildah/stable:v1.17.0
      workingDir: $(workspaces.source.path)
      script: |
        buildah --storage-driver=$(params.STORAGE_DRIVER) bud \
          --no-cache -f $(params.DOCKERFILE) -t $(params.IMAGE) .
      volumeMounts:
        - name: varlibcontainers
          mountPath: /var/lib/containers
      securityContext:
        privileged: true
    - name: push
      image: quay.io/buildah/stable:v1.17.0
      workingDir: $(workspaces.source.path)
      script: |
        buildah --storage-driver=$(params.STORAGE_DRIVER) push \
          --creds ${USERNAME}:${PASSWORD} $(params.IMAGE)
      volumeMounts:
        - name: varlibcontainers
          mountPath: /var/lib/containers
      securityContext:
        privileged: true
      env:
        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: $(params.IMAGE_PUSH_SECRET_NAME)
              key: username
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: $(params.IMAGE_PUSH_SECRET_NAME)
              key: password
  volumes:
    - name: varlibcontainers
      emptyDir: {}

```

For authentication we simply refer to a secret store in our NS

#### Pipeline:

``` yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build/push pipeline
spec:
  workspaces:
    - name: myworkspace
  tasks:
    - name: fetch-repository
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: myworkspace
      params:
        - name: url
          value: {YOUR_GIT_REPOSITORY_URL}
        - name: deleteExisting
          value: "true"
    - name: build
      taskRef:
        name: buildah
      runAfter:
        - fetch-repository
      params:
        - name: IMAGE
          value: docker.io/{YOUR_DOCKERHUB_REPOSITORY_NAME}
        - name: IMAGE_PUSH_SECRET_NAME
          value: image-push-secrets
      workspaces:
        - name: source
          workspace: myworkspace
```
### Secret creation:

```  bash
export USERNAME={QUAY USER}
export PASSWORD={QUAY PASS}
kubectl create secret generic image-push-secrets — from-literal username=$USERNAME — from-literal password=$PASSWORD
```