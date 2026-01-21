# Create an Event Listener and Triggers Logic

After completing this lab, you will be able to:

- Create an EventListener, a TriggerBinding and a TriggerTemplate
- State how to trigger a deployment when changes are made to GitHub

Clone the first lab with the task and the pipeline

```sh
git clone https://github.com/agomezr/ttwst-jhxyb-ci-cd-pipeline_js.git
```

Go to the lab folder

```sh
cd ttwst-jhxyb-ci-cd-pipeline_js/labs/02_add_git_trigger/
```

Install everything from the previous lab.

```sh
kubectl apply -f tasks.yaml
kubectl apply -f pipeline.yaml
```

Check everything is ok

```sh
tkn task ls
tkn pipeline ls
kubectl get task
kubectl get pipeline
```

## TriggerBinding and TriggerTemplate

You will update the eventlistener.yaml file to define an EventListener named cd-listener that references a TriggerBinding named cd-binding and a TriggerTemplate named cd-template.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: cd-listener
spec:
  serviceAccountName: pipeline
  triggers:
    - bindings:
      - ref: cd-binding
      template:
        ref: cd-template
```

The serviceAccountName is a “user/role” for kubernetes clúster. If not exists or does not have the permissions (RBAC) a forbidden (403) will be set in the logs.

Apply the EventListener resource to the cluster:

```sh
kubectl apply -f eventlistener.yaml
```

Check that it was created correctly.

```sh
kubectl get eventlistener
tkn eventlistener ls
```

Next, you need a way to bind the incoming data from the event to pass on to the pipeline. To accomplish this, you use a TriggerBinding.

Update the triggerbinding.yaml file to create a TriggerBinding named cd-binding that takes the body.repository.url and body.ref and binds them to the parameters repository and branch, respectively.

- The first thing you want to do is give the TriggerBinding the same name that is referenced in the EventListener, which is cd-binding.
- Next, you need to add a parameter named repository to the spec: section with a value that references $(body.repository.url).
- Finally, you need to add a parameter named branch to the spec: section with a value that references $(body.ref).

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: cd-binding
spec:
  params:
    - name: repository
      value: $(body.repository.url)
    - name: branch
      value: $(body.ref)
```

Apply the new TriggerBinding definition to the cluster:

```sh
kubectl apply -f triggerbinding.yaml
```

Check if it was created

```sh
kubectl get triggerbinding
tkn triggerbinding list
```

The TriggerTemplate takes the parameters passed in from the TriggerBinding and creates a PipelineRun to start the pipeline.

- The first thing you want to do is give the TriggerTemplate the same name that is referenced in the EventListener, which is cd-template.
- Next, you need to add a parameter named repository to the spec: section with a description: of "The git repo" and a default: of " ".
- Then, you need to add a parameter named branch to the spec: section with a description: of "the branch for the git repo" and a default: of master.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: cd-template
spec:
  params:
    - name: repository
      description: The git repo
      default: " "
    - name: branch
      description: the branch for the git repo
      default: master
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: cd-pipeline-run-
      spec:
        serviceAccountName: pipeline
        pipelineRef:
          name: cd-pipeline
        params:
          - name: repo-url
            value: $(tt.params.repository)
          - name: branch
            value: $(tt.params.branch)
```

Note that while the parameter you bound from the event is a repository, you pass it on as repo-url to the pipeline. This is to show that the names do not have to match, allowing you to use any pipeline to map parameters into. The `tt.params.<something>` is for TriggerTemplate params.

Apply the new TriggerTemplate definition to the cluster:

```sh
kubectl apply -f triggertemplate.yaml
```

Check if it was created

```sh
kubectl get triggertemplate
tkn triggertemplate list
```

Now it is time to call the event listener and start a PipelineRun. You can do this locally using the curl command to test that it works.

Notice that the call is a endpoint with body parameters (JSON) to the pipeline. We need to terminals: one for call the pipeline via curl and the other to start a server and listen waiting for the trigger.

### Start server

You need to run the kubectl port-forward command to forward the port for the event listener so that you can call it on localhost.

Use the kubectl port-forward command to forward port 8090 to 8080.

```sh
kubectl port-forward service/el-cd-listener  8090:8080
```

### Trigger event listener

Use the curl command to send a payload to the event listener service.

```sh
curl -X POST http://localhost:8090 \
  -H 'Content-Type: application/json' \
  -d '{"ref":"main","repository":{"url":"https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js"}}
```

This should start a PipelineRun. You can check on the status with this command:

```console
$ tkn pipelinerun ls
NAME                    STARTED          DURATION   STATUS
cd-pipeline-run-lzxth   25 seconds ago   ---        Running
```

You can also examine the PipelineRun logs using this command (the -L means "last" so that you do not have to look up the name for the last run):

```console
$ tkn pipelinerun logs --last
[clone : checkout] Cloning into 'ttwst-jhxyb-ci-cd-pipeline_js'...
[lint : echo-message] Calling ESLint linter...
[tests : echo-message] Running unit tests with Jest...
[build : echo-message] Building image for https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js ...
[deploy : echo-message] Deploying main branch of https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js ...
```
