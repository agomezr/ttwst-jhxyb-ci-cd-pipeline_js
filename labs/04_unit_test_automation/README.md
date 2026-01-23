# Testing in the Pipeline

After completing this lab, you will be able to:

- Create a custom ESLint task to install the eslint task
- Describe the parameters required to use the eslint task
- Use the eslint task in a Tekton pipeline to lint your JavaScript code
- Create a test task from scratch and use it in your pipeline

Clone the repository and change folder

```sh
git clone https://github.com/agomezr/ttwst-jhxy
cd ttwst-jhxyb-ci-cd-pipeline_js/labs/04_unit_test_automation/
```

Install the tack to your cluster before proceeding

```sh
kubectl apply -f tasks.yaml
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
```

Check the install

```console
$ tkn task ls
NAME        DESCRIPTION              AGE
cleanup     This task will clea...   13 seconds ago
echo                                 13 seconds ago
git-clone   These Tasks are Git...   11 seconds ago
```

You also need a PersistentVolumeClaim (PVC) to use as a workspace.

```console
$ kubectl apply -f pvc.yaml
persistentvolumeclaim/pipelinerun-pvc created
```

You can now reference this persistent volume claim by its name pipelinerun-pvc when creating workspaces for your Tekton tasks.

## Step 0: Check for cleanup

Please check as part of Step 0 for the new cleanup task that has been added to tasks.yaml file.

When a task causes a compilation of the JavaScript code, it leaves behind build files and node_modules that are owned by the specific user. For consecutive pipeline runs, the git-clone task tries to empty the directory but needs privileges to remove these files, and this cleanup task takes care of that.

The init task is added to the pipeline.yaml file, which runs every time before the clone task.

## Step 1: Add the ESLint task

Your pipeline has a placeholder for a lint step that uses the echo task. Now, it is time to replace it with a real linter.

You are going to use ESLint to lint your JavaScript code. Since there isn't a pre-built ESLint task in Tekton Catalog, you will create your own ESLint task.

Add a new task called eslint to the tasks.yaml file. Remember, each new task must be separated using three dashes—on a separate line.

The task should:
- Have a workspace named source
- Accept parameters for image (default: node:18-alpine) and args
- Run ESLint with the specified arguments

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: eslint
spec:
  workspaces:
    - name: source
  steps:
    - name: install-dependencies
      image: node:18
      workingDir: $(workspaces.source.path)
      script: |
        npm install
    - name: run-eslint
      image: node:18
      workingDir: $(workspaces.source.path)
      script: |
        npx eslint .
```

Apply these changes to your cluster:

```sh
kubectl apply -f tasks.yaml
```

## Step 2: Modify the pipeline to use ESLint

- Add the workspaces: keyword to the lint task after the task name: but before the taskRef:
- Specify the workspace name: as source
- Specify the workspace: reference as pipeline-workspace
- Change the taskRef: from echo to reference the eslint task
- Change the parameters to use image and args instead of message

```yaml
- name: lint
      workspaces:
        - name: source
          workspace: pipeline-workspace
      taskRef:
        name: eslint
      params:
      - name: image
        value: "node:18-alpine"
      - name: args
        value: ["--format", "stylish", "--ext", ".js", "."]
      runAfter:
        - clone
    # Note: The remaining tasks are unchanged
```

Apply these changes to your cluster:

```console
$ kubectl apply -f pipeline.yaml
pipeline.tekton.dev/cd-pipeline created
```

## Step 3: Run the pipeline

```sh
tkn pipeline start cd-pipeline \
    -p repo-url="https://github.com/agomezr/ttwst-jhxyb-ci-cd-pipeline_js" \
    -p branch="main" \
    -w name=pipeline-workspace,claimName=pipelinerun-pvc \
    --showlog
```

## Step 4: Create a test task

Your pipeline also has a placeholder for a tests task that uses the echo task. Now, you will replace it with real unit tests. In this step, you will replace the echo task with a call to a unit test framework called Jest.

There are no tasks in the Tekton Catalog for Jest, so you will write your own.

Update the tasks.yaml file adding a new task called jest that uses the shared workspace for the pipeline and runs npm test in a node:18-alpineimage.

Here is a bash script to install the Node.js dependencies and run the Jest tests. You can use this as the shell script in your new task:

```sh
#!/bin/sh
set -e
npm ci
npm test
```

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: jest
spec:
  description: This task runs Jest tests for JavaScript applications
  workspaces:
    - name: source
  params:
    - name: args
      description: Arguments to pass to Jest
      type: string
      default: "--verbose"
  steps:
    - name: test
      image: node:18-alpine
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        set -e
        npm ci
        npm test -- $(params.args)
```

Apply these changes to your cluster:

```console
$ kubectl apply -f tasks.yaml
task.tekton.dev/echo configured
task.tekton.dev/cleanup configured
task.tekton.dev/eslint configured
task.tekton.dev/jest created
```

## Step 5: Modify the pipeline to use Jest

The final step is to use the new jest task in your existing pipeline in place of the echo task placeholder.

```yaml
- name: tests
      workspaces:
        - name: source
          workspace: pipeline-workspace
      taskRef:
        name: jest
      params:
      - name: args
        value: "--verbose --coverage"
      runAfter:
        - lint
```

Apply these changes to your cluster:

```console
$ kubectl apply -f pipeline.yaml
pipeline.tekton.dev/cd-pipeline configured
```

## Step 6: Run the pipeline again

Now that you have your tests task complete, run the pipeline again using the Tekton CLI to see your new test tasks run:

```sh
tkn pipeline start cd-pipeline \
    -p repo-url="https://github.com/ibm-developer-skills-network/ttwst-jhxyb-ci-cd-pipeline_js.git" \
    -p branch="main" \
    -w name=pipeline-workspace,claimName=pipelinerun-pvc \
    --showlog
```

You can see the pipeline run status by listing the PipelineRun in other terminal with:

```console
$ tkn pipelinerun ls
NAME                    STARTED          DURATION   STATUS
cd-pipeline-run-qjx4s   1 minute ago     ---        Running
cd-pipeline-run-2rdcq   15 minutes ago   14m42s     Failed
```

You can check the logs of the last run with:

```sh
tkn pipelinerun logs --last
```