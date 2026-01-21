# Create a Workspace, use git-clone from Tkn Hub

Clone the repo of GitHub

```sh
git clone https://github.com/agomezr/ttwst-jhxyb-ci-cd-pipeline_js.git
```

Change directory
```sh
cd ttwst-jhxyb-ci-cd-pipeline_js/labs/03_use_tekton_catalog/
```

Apply the tasks to Kubernetes ecosystem

```sh
kubectl apply -f tasks.yaml
```

You start by finding a task to replace the checkout task you initially created. While it was OK as a learning exercise, it needs a lot more capabilities to be more robust, and it makes sense to use the community-supplied task instead.

## Add the git-clone task from Tekton Catalog

You can browse the Tekton Catalog, find the git-clone yaml file, copy the URL to the yaml file, and use kubectl to apply it manually.

Use this command to apply the official Tekton Catalog task manifest for git-clone to your Kubernetes cluster using kubectl. This installs the git-clone task into your cluster under your current active namespace.

```sh
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
```

To check that the task is apply we need to run a CLI command because no file-check is possible

```console
$ kubectl get tasks
NAME        AGE
checkout    4h55m
echo        4h55m
git-clone   3m37s
```

## Create a Workspace

Viewing the git-clone task requirements, you see that while it supports many more parameters than your original checkout task, it only requires two things:

- The URL of a Git repo to clone, provided with the url param
- A workspace called output

To see the code in terminal

```sh
kubectl get task git-clone -o yaml
```

You start by creating a PersistentVolumeClaim (PVC) to use as the workspace:

A workspace is a disk volume that can be shared across tasks. The way to bind to volumes in Kubernetes is with PersistentVolumeClaim.

Since creating PVCs is beyond the scope of this lab, you have been provided with the following pvc.yaml file with these contents:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pipelinerun-pvc
spec:
  storageClassName: skills-network-learner
  resources:
    requests:
      storage:  1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

Apply the new task to the cluster

```console
$ kubectl apply -f pvc.yaml
persistentvolumeclaim/pipelinerun-pvc created
```

You can now reference this persistent volume by its name pipelinerun-pvc when creating workspaces for your Tekton tasks.

## Add a workspace to the pipeline

In this step, you will add a workspace to the pipeline using the persistent volume claim you just created. To do this, you will edit the pipeline.yaml file and add a workspaces: definition as the first line under the spec: but before params: and call it pipeline-workspace. Then, you will add the workspace to the pipeline clone task and change the task to reference git-clone instead of your checkout task.

```yaml
spec:
  workspaces:
    - name: pipeline-workspace
  params:
    - name: repo-url
    - name: branch
      default: "master"
  tasks:
    - name: clone
      workspaces:
        - name: output
          workspace: pipeline-workspace
      taskRef:
        name: git-clone
      params:
      - name: url
        value: $(params.repo-url)
      - name: revision
        value: $(params.branch)
    # Note: The remaining tasks are unchanged. Do not delete them.
```

Apply the pipeline to your cluster:

```console
$ kubectl apply -f pipeline.yaml
pipeline.tekton.dev/cd-pipeline created
```

## Run the Pipeline

You can now use the Tekton CLI (tkn) to create PipelineRun to run the pipeline.

Use the following command to run the pipeline, passing in the URL of the repository, the branch to clone, the workspace name, and the persistent volume claim name.

```console
$ tkn pipeline start cd-pipeline \
    -p repo-url="https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js.git" \
    -p branch="main" \
    -w name=pipeline-workspace,claimName=pipelinerun-pvc \
    --showlog
PipelineRun started: cd-pipeline-run-62q4r
Waiting for logs to be available...
```

You can always see the pipeline run status by listing PipelineRuns with:

```console
$ tkn pipelinerun ls
NAME                    STARTED        DURATION   STATUS
cd-pipeline-run-62q4r   1 minute ago   32s
```

You can check the logs of the last run with:

```sh
tkn pipelinerun logs --last
```
