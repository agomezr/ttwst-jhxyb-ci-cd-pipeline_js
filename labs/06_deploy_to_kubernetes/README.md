# Deploy to Kubernetes

You are now at the deploy step, which is the last step in your CD pipeline. For this step, you will use the OpenShift client to deploy your Docker image to an OpenShift cluster. 

OpenShift is based on Kubernetes. Anything you can do with Kubernetes, you can do that and more with OpenShift. This lab uses the commands kubectl and oc interchangeably because oc is a proper superset of kubectl. 

After completing this lab, you will be able to:

Determine if the openshift-client ClusterTask is available on your cluster
Describe the parameters required to use the openshift-client ClusterTask
Use the openshift-client ClusterTask in a Tekton pipeline to deploy your Docker image to Kubernetes

Clone the code repo and change the active folder in the terminal

```sh
git clone https://github.com/agomezr/ttwst-jhxyb-ci-cd-pipeline_js.git
cd ttwst-jhxyb-ci-cd-pipeline_js/labs/06_deploy_to_kubernetes/
```

Install the tasks and the Persistent Volume Clain

```sh
kubectl apply -f tasks.yaml
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
kubectl apply -f pvc.yaml
```

Check the installation

```console
$ tkn task ls
NAME        DESCRIPTION              AGE
cleanup     This task will clea...   18 seconds ago
echo                                 18 seconds ago
eslint                               18 seconds ago
git-clone   These Tasks are Git...   9 seconds ago
jest        This task runs Jest...   17 seconds ago
```

## Step 1: Check for the openshift-client ClusterTask

Your pipeline currently has a placeholder for a deploy step that uses the echo task. Now, it is time to replace it with a real deployment.

To deploy to OpenShift, this lab uses the openshift-client task, which is already available in the OpenShift environment as a ClusterTask. Since ClusterTasks are installed cluster-wide by an administrator, they can be referenced directly in pipelines without any additional installation.

Check that the openshift-client task is installed as a ClusterTask using the command kubectl get tasks.

```console
$ tkn clustertask ls
NAME               DESCRIPTION              AGE
openshift-client   This task runs comm...   32 weeks ago
...
```

## Step 2: Reference the openshift-client task

First you need to update the pipeline.yaml file to use the new openshift-client task.

You must now reference the new openshift-client ClusterTask that you want to use in the deploy pipeline task.

In the previous steps, you simply changed the name of the reference to the task, but since the openshift-client task is installed as a ClusterTask, you need to add the statement kind: ClusterTask under the name so that Tekton knows to look for a ClusterTask and not a regular Task.

## Step 3: Update the task parameters

The documentation for the openshift-client task details that there is a parameter named SCRIPTthat you can use to run oc commands. Any command you can use with kubectl can also be used with oc. This is what you will use to deploy your image.

The command to deploy an image on OpenShift is:

```sh
oc create deployment {name} --image={image-name}
```

Since you might want to reuse this pipeline to deploy different applications, you should make the deployment name a parameter that can be passed in when the pipeline runs. You already have the image name as a parameter from the build task that you can use.

## Step 4: Update the pipeline parameters

Now that you are passing in the app-name parameter to the deploy task, you need to go back to the top of the pipeline.yaml file and add the parameter there so that it can be passed into the pipeline when it is run.

If you changed everything correctly, the full deploy task in the pipeline should look like this:

```yaml
- name: deploy
      taskRef:
        name: openshift-client
        kind: ClusterTask
      params:
      - name: SCRIPT
        value: "oc create deploy $(params.app-name) --image=$(params.build-image)"
      runAfter:
        - build
```

Also, the full parameter list for your pipeline should look like this:

```yaml
spec:
  params:
    - name: app-name
    - name: build-image
    - name: repo-url
    - name: branch
      default: master
```

Before you proceed with running commands in the terminal, make sure that you are in the …/06_deploy_to_kubernetes/ folder.

## Step 6: Apply changes and run the pipeline

Apply the same changes you just made to pipeline.yaml to your cluster:

```console
$ kubectl apply -f pipeline.yaml
pipeline.tekton.dev/cd-pipeline created
```

When you start the pipeline, you now need to pass in the app-name parameter, which is the name of the application to deploy.

Your application is called hitcounter so this is the name that you will pass in, along with all the other parameters from the previous steps.

Now, start the pipeline to see your new deploy task run. Use the Tekton CLI pipeline start command to run the pipeline, passing in the parameters repo-url, branch, app-name, and build-image using the -p option. Specify the workspace pipeline-workspace and persistent volume claim pipelinerun-pvc using the -w option:

```sh
tkn pipeline start cd-pipeline \
    -p repo-url="https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js.git" \
    -p branch=main \
    -p app-name=hitcounter \
    -p build-image=image-registry.openshift-image-registry.svc:5000/$SN_ICR_NAMESPACE/tekton-lab:latest \
    -w name=pipeline-workspace,claimName=pipelinerun-pvc \
    --showlog
```

The process will end in more than 15 minutes

You can see the pipeline run status in other terminal by listing the pipeline runs with:

```console
$ tkn pipelinerun ls
NAME                    STARTED         DURATION     STATUS
cd-pipeline-run-fbxbx   1 minute ago    59 seconds   Succeeded
```

You can check the logs of the last run with:

```sh
tkn pipelinerun logs --last
```

If it is successful, the last line you should see in the logs is:

```sh
[deploy : oc] deployment.apps/hitcounter created
```

## Step 7: Check the deployment

Now, check to see if the deployment is running. Use the kubectl command to check that your deployment is in a running state.

```console
$ kubectl get all -l app=hitcounter
NAME                              READY   STATUS    RESTARTS   AGE
pod/hitcounter-7c9f95784d-rk4tf   1/1     Running   0          2m46s
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hitcounter   1/1     1            1           2m46s
NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/hitcounter-7c9f95784d   1         1 
```

If your pod is running, your application has been successfully deployed.
