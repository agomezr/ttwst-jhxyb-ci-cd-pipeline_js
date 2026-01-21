# Create a base pipeline

This folder holds the files for the lab _Create a Base Pipeline_ which is part of the **IBM-CD0215EN-Skills Network Introduction to CI/CD** course.

## Create a task.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-world
spec:
  steps:
    - name: echo
      image: alpine:3
      command: [/bin/echo]
      args: ["Hello World!"]
```

Apply it to the Kubernetes cluster

```sh
kubectl apply -f tasks.yaml
```

Chek that the task were created

```sh
kubectl get tasks  # Kubernetes native command to list
tkn task ls # Tekton CLI command to list
```

Create a hello-pipeline pipeline

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: hello-pipeline
spec:
  tasks:
    - name: hello
      taskRef:
        name: hello-world
```

Apply it to the Kubernetes cluster, every time the file update an apply must be done

```sh
kubectl apply -f pipeline.yaml
```

Chek that the pipeline were created

```sh
tkn pipeline ls # Tekton CLI command to list
```

Run the pipeline using the Tekton CLI

```sh
tkn pipeline start --showlog hello-pipeline
```

The output of the command is

```sh
PipelineRun started: hello-pipeline-run-9vkbb
Waiting for logs to be available...
[hello : echo] Hello World!
```

Update the task.yaml to include a param that flows to the pipeline

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: echo
spec:
  params:
    - name: message
      description: The message to echo
      type: string
  steps:
    - name: echo-message  
      image: alpine:3
      command: [/bin/echo]
      args: ["$(params.message)"]
```

Apply the new task definition to the cluster:

```sh
kubectl apply -f tasks.yaml
```

Propagate the change for parameter in the pipeline.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: hello-pipeline
spec:
  params:
    - name: message
  tasks:
    - name: hello
      taskRef:
        name: echo
      params:
        - name: message
          value: "$(params.message)"
```

Apply it to the cluster

```sh
kubectl apply -f pipeline.yaml
```

Test it in the CLI

```sh
tkn pipeline start hello-pipeline \
    --showlog  \
    -p message="Hello Tekton!"
```

Output

```sh
PipelineRun started: hello-pipeline-run-9qf42
Waiting for logs to be available...
[hello : echo-message] Hello Tekton!
```

You can have multiple definitions in a single YAML file by separating them with three dashes --- on a single line.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: echo
spec:
  params:
    - name: message
      description: The message to echo
      type: string
  steps:
    - name: echo-message  
      image: alpine:3
      command: [/bin/echo]
      args: ["$(params.message)"]
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: checkout
spec:
  params:
    - name: repo-url
      description: The URL of the git repo to clone
      type: string
    - name: branch
      description: The branch to clone
      type: string
  steps:
    - name: checkout
      image: bitnami/git:latest
      command: [git]
      args: ["clone", "--branch", "$(params.branch)", "$(params.repo-url)"]
```

Add to cluster

```sh
kubectl apply -f tasks.yaml
```

Output 

```sh
task.tekton.dev/echo configured
task.tekton.dev/checkout created
```

Finally, you will create a pipeline called cd-pipeline to be the starting point of your Continuous Delivery pipeline.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: hello-pipeline
spec:
  params:
    - name: message
  tasks:
    - name: hello
      taskRef:
        name: echo
      params:
        - name: message
          value: "$(params.message)"
---
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: cd-pipeline
spec:
  params:
    - name: repo-url
    - name: branch
      default: "master"
  tasks:
    - name: clone
      taskRef:
        name: checkout
      params:
      - name: repo-url
        value: "$(params.repo-url)"
      - name: branch
        value: "$(params.branch)"
```

Apply to cluster again

```sh
kubectl apply -f pipeline.yaml
```

Output. Notice that the previous pipeline is marked as configured instead of created

```sh
pipeline.tekton.dev/hello-pipeline configured
pipeline.tekton.dev/cd-pipeline created
```

Test it in the CLI

```sh
tkn pipeline start cd-pipeline \
    --showlog \
    -p repo-url="https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js.git" \
    -p branch="main"
```

Output

```sh
PipelineRun started: cd-pipeline-run-rf6zp
Waiting for logs to be available...
[clone : checkout] Cloning into 'ttwst-jhxyb-ci-cd-pipeline_js'...
```

Create a pipeline task for each of these:

| Task name |	Build after	| Message |
| --- | --- | --- |
| lint	| clone |	Calling ESLint linter… |
| tests |	lint	| Running unit tests with Jest… |
| build	| tests	| Building image for $(params.repo-url) … |
| deploy	| build	| Deploying $(params.branch) branch of $(params.repo-url) … |

The code of the pipeline.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: cd-pipeline
spec:
  params:
    - name: repo-url
    - name: branch
      default: "master"
  tasks:
    - name: clone
      taskRef:
        name: checkout
      params:
      - name: repo-url
        value: "$(params.repo-url)"
      - name: branch
        value: "$(params.branch)"
    - name: lint
      taskRef:
        name: echo
      params:
      - name: message
        value: "Calling ESLint linter..."
      runAfter:
        - clone
    - name: tests
      taskRef:
        name: echo
      params:
      - name: message
        value: "Running unit tests with Jest..."
      runAfter:
        - lint
    - name: build
      taskRef:
        name: echo
      params:
      - name: message
        value: "Building image for $(params.repo-url) ..."
      runAfter:
        - tests
    - name: deploy
      taskRef:
        name: echo
      params:
      - name: message
        value: "Deploying $(params.branch) branch of $(params.repo-url) ..."
      runAfter:
        - build
```

Apply it

```sh
kubectl apply -f pipeline.yaml
```

Run the pipeline using the Tekton CLI:

```sh
tkn pipeline start cd-pipeline \
    --showlog \
    -p repo-url="https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js.git" \
    -p branch="main"
```

Output

```sh
PipelineRun started: cd-pipeline-run-x8zjx
Waiting for logs to be available...
[clone : checkout] Cloning into 'CI-CD_Actions'...

[lint : echo-message] Calling ESLint linter...

[tests : echo-message] Running unit tests with Jest...

[build : echo-message] Building image for https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js.git ...

[deploy : echo-message] Deploying main branch of https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js.git...
```
