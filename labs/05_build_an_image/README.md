# Building an image

After completing this lab, you will be able to:

- Determine which ClusterTasks are available on your cluster
- Describe the parameters required to use the buildah ClusterTask
- Use the buildah ClusterTask in a Tekton pipeline to build an image and push it to an image registry

Clone the repository and change folder

```sh
git clone https://github.com/agomezr/ttwst-jhxyb-ci-cd-pipeline_js.git
cd ttwst-jhxyb-ci-cd-pipeline_js/labs/05_build_an_image/
```

Install the tack to your cluster before proceeding

```sh
kubectl apply -f tasks.yaml
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
```

Check that you have all of the previous tasks installed

```console
$ tkn task ls
NAME        DESCRIPTION              AGE
cleanup     This task will clea...   3 minutes ago
echo                                 3 minutes ago
eslint                               3 minutes ago
git-clone   These Tasks are Git...   4 minutes ago
jest        This task runs Jest...   3 minutes ago
```

## Step 1: Use the buildah ClusterTask

Your pipeline currently has a placeholder for a build step that uses the echo task. Now it is time to replace it with a real image builder.

To build container images, this lab uses the buildah task, which is already available in the OpenShift environment as a ClusterTask. Since ClusterTasks are installed cluster-wide by an administrator, they can be referenced directly in pipelines without any additional installation.

OpenShift is Red Hat's enterprise application platform, built on top of Kubernetes.
Although it uses pure Kubernetes for orchestration underneath, OpenShift adds essential layers for the enterprise environment.

In this lab, you will reference the existing buildah ClusterTask when updating your pipeline configuration.

## Step 2: Add a workspace to the pipeline task

Now, you will update the pipeline.yaml file to use the new buildah task.

While reading the documentation for the buildah task, you will notice that it requires a workspace named source.

Add the workspace to the build task after the name but before the taskRef. The workspace that you have been using is named pipeline-workspace and the name the task requires is source.

## Step 3: Reference the buildah task

Now, you need to reference the new buildah task that you want to use. In the previous steps, you simply changed the name of the reference to the task. But since the buildah task is a ClusterTask, you need to add the statement kind: ClusterTask under the name so that Tekton knows to look for a ClusterTask and not a regular Task.

## Step 4: Update the task parameters

The documentation for the buildah task details several parameters, but only one of them is required. You need to use the IMAGE parameter to hold the name of the image you want to build.

Since you might want to reuse this pipeline to build different images, you will make it a variable parameter that can be passed in when the pipeline runs. To do this, you need to change it here and add a parameter to the pipeline itself.

Change the message parameter to IMAGE and specify the value of $(params.build-image):

Now that you are passing in the IMAGE parameter to this task, you need to go back to the top of the pipeline.yaml file and add the parameter there so that it can be passed into the pipeline when it is run.

Add a parameter named build-image to the existing list of parameters at the top of the pipeline under spec.params.

## Step 5: Check your work

If you changed everything correctly, the full build task in the pipeline should look like this:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:  
  name: cd-pipeline
spec:
  workspaces:
    - name: pipeline-workspace
  params:
    - name: build-image
    - name: repo-url
    - name: branch
      default: master
  tasks:
//… some code
- name: build
      workspaces:
        - name: source
          workspace: pipeline-workspace
      taskRef:
        name: buildah
        kind: ClusterTask
      params:
      - name: IMAGE
        value: "$(params.build-image)"
      runAfter:
        - tests
//… some code
```

## Step 6: Apply changes and run the pipeline

Apply the same changes you just made to pipeline.yaml to your cluster:

```console
$ kubectl apply -f pipeline.yaml
pipeline.tekton.dev/cd-pipeline created
```

Next, make sure that the persistent volume claim for the workspace exists by applying it using kubectl:

```console
$ kubectl apply -f pvc.yaml
persistentvolumeclaim/pipelinerun-pvc created
```

When you start the pipeline, you need to pass in the build-image parameter, which is the name of the image to build.

This will be different for every learner that uses this lab. Here is the format:

image-registry.openshift-image-registry.svc:5000/$SN_ICR_NAMESPACE/tekton-lab:latest

Notice the variable $SN_ICR_NAMESPACE in the image name. This is automatically set to point to your container namespace.

Now, start the pipeline to see your new build task run. Use the Tekton CLI pipeline start command to run the pipeline, passing in the parameters repo-url, branch, and build-image using the -p option. Specify the workspace pipeline-workspace and volume claim pipelinerun-pvc using the -w option:

```sh
tkn pipeline start cd-pipeline \
    -p repo-url="https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js.git" \
    -p branch=main \
    -p build-image=image-registry.openshift-image-registry.svc:5000/$SN_ICR_NAMESPACE/tekton-lab:latest \
    -w name=pipeline-workspace,claimName=pipelinerun-pvc \
    --showlog
```

You should see Waiting for logs to be available... while the pipeline runs. The logs will be shown on the screen. Wait until the pipeline run completes successfully.

You can see the pipeline run status by listing the pipeline in other terminal, runs with:

```console
$ tkn pipelinerun ls
NAME                    STARTED          DURATION   STATUS
cd-pipeline-run-fx82d   57 seconds ago   ---        Running
```

You can check the logs of the last run with:

```console
$ tkn pipelinerun logs --last
Pipeline still running …
```
