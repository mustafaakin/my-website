---
title: Using Jsonnet to Generate Dynamic Tekton Pipelines in Kubernetes
date: 2020-04-26
--- 

For the readers that might be unaware, [Tekton](https://github.com/tektoncd/pipeline) is a cloud-native CI/CD solution. In other words, it's a pipeline for Kubernetes that can generate DAGs (directed acyclic graph). It's one of the most straightforward solutions to run dependent Pods and steps in Kubernetes. Tekton runs pipelines as Task CRDs (Custom Resource Definition), whereas every task can have one or more sequential steps. Also, a Task can have a dependency on another task, which allows building a DAG. Also, a Task and Pipeline can be parameterized, which allows reusing components.

In this blog post, I will take it one step further and introduce you to a methodology to generate more dynamic tasks, with the help of Jsonnet. [Jsonnet](https://jsonnet.org/) is a data templating language by Google, which looks like JSON, but also allows conditionals, iterations, methods to enable writing cleaner files. Although JSON is human-readable, it's not easily human writable when you want to lots of stuff. And also, templating Kubernetes with YAML is an abomination that I never like or accept, and tools that embrace Jsonnet makes it easy to template Kubernetes configuration files. [Ksonnet](https://github.com/ksonnet/ksonnet) is an excellent example. But you might also ask, "why the hell am I templating too much configuration for Kubernetes, I just wanted to deploy my microservice," but that's another topic that can start flamewars.

Tekton consists of [Task](https://github.com/tektoncd/pipeline/blob/master/docs/tasks.md), [TaskRun](https://github.com/tektoncd/pipeline/blob/master/docs/taskruns.md), [Pipeline](https://github.com/tektoncd/pipeline/blob/master/docs/pipelines.md), and [PipelineRun](https://github.com/tektoncd/pipeline/blob/master/docs/pipelineruns.md) objects. Tasks define the steps. Pipeline define which tasks are going to be used. PipelineRun is a Pipeline instance that you want to execute, possibly with multiple parameters as input. As we are going to use Jsonnet to generate dynamic PipelineRun's procedurally, we are not going to use Tasks, but embed the Task spec to PipelineRun itself. The following is an example of a Pipeline with embedded Tasks:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: embed-test
spec:
  pipelineSpec:
    tasks:
      - name: task-1
        taskSpec:
          steps:
            - name: build
              image: ubuntu
              script: echo "Compiling beep boop..."
            - name: test
              image: ubuntu
              script: echo "Compiled, running tests.."
      - name: task-2
        taskSpec:
          steps:
            - name: echo
              image: ubuntu
              script: echo "This is another Pod!"                
```

As you can probably imagine, the task-1 and task-2 is a Pod, and Tekton injects wrappers to execute the steps sequentially. The above is 23 lines of YAML for 3 echo messages. Of course, it's not a realistic task, but let's see how we can reduce that with Jsonnet.

The `tekton.jsonnet` file to simplify boilerplate:

```javascript
{
  step: function(name, image, script) {
    name: name,
    image: image,
    script: script,
  },
  task: function(name, steps) {
    name: name,
    taskSpec: {
      steps: steps,
    },
  },
  pipeline: function(name, tasks) {
    apiVersion: 'tekton.dev/v1beta1',
    kind: 'PipelineRun',
    metadata: {
      name: name,
    },
    spec: {
      pipelineSpec: {
        tasks: tasks,
      },
    },
  },
}
```

The actual pipeline generator which you can execute with `jsonnet example.jsonnet | kubectl apply -f -`

```javascript
local tkn = import 'tekton.jsonnet';

local task1 = tkn.task(name='task-1', steps=[
  tkn.step('compile', 'ubuntu', "echo 'Compling beep boop...'"),
  tkn.step('tests', 'ubuntu', "echo 'Compiled, running tests...'"),
]);

local task2 = tkn.task(name='task-2', steps=[
  tkn.step('run', 'ubuntu', "echo 'This is another Pod!'"),
]);

tkn.pipeline(name='embed-test-jsonnet', tasks=[task1, task2])
```

It looks cleaner in this version because it's smaller, and unlike YAML, it does not require indentation to be valid. The example just shortened writing the Kubernetes-specific configuration and provided helper functions. The above example is simple, not worth to introduce another language. In what conditions using Jsonnet would thrive? The power of Jsonnet comes from helping you to modularize the code by using imports, iterations, conditionals, and some of the [standard library functions](https://jsonnet.org/ref/stdlib.html). Note that Tekton already supports parameters and [conditional execution](https://github.com/tektoncd/pipeline/blob/master/docs/conditions.md) and it allows acting based on the results of the tasks.

In the following (somewhat) realistic example, we'll clone my favorite Java library from Github, [Failsafe](https://github.com/jhalterman/failsafe.git) repository with given branch and execute tests for multiple versions of Java in parallel. Note that we will not be using Tekton PipelineResource CRD, because they are still in Alpha, and the devs don't [feel comfortable updating](https://github.com/tektoncd/pipeline/blob/master/docs/resources.md#why-arent-pipelineresources-in-beta) and also cloning from the git is simple step to write. The YAML equivalent would look like the following:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: git-test
spec:
  pipelineSpec:
    tasks:
      - name: java-11
        taskSpec:
          steps:
            - name: echo
              image: alpine/git:v2.24.2
              script: git clone https://github.com/jhalterman/failsafe.git .
            - name: test
              image: maven:3.6.3-jdk-11
              script: mvn clean test -T1C -B
```

So, to test it with multiple versions of Java in parallel, we need to duplicate the tasks for each Java version and duplicate 11 lines for each. Let's see how we can simplify it with Jsonnet:

```javascript
local tkn = import 'tekton.jsonnet';

local git = function(repo, branch)
  tkn.step(
    name='git-clone',
    // Always pin your versions in Docker images to avoid unnecessary pulls
    // You can achieve more security with sha256 pinning
    image='alpine/git:v2.24.2',
    // Single branch and depth=1 makees checking out much faster for large repos
    script='git clone %s --depth=1 --single-branch --branch %s .' % [repo, branch]
  );


local mvnTest = function(version)
  tkn.task(
    name='java-' + version,
    steps=[
        // My favorite Java library <3
      git('https://github.com/jhalterman/failsafe.git', 'master'),
      // Batch mode causes less output in Maven
      tkn.step(name='test', image='maven:' + version, script='mvn test -T1C --batch-mode'),
    ]
  );

local versions = [
  "3-jdk-8",
  "3-jdk-11",
  "3-jdk-14",
];

tkn.pipeline(
  name='java-test-jsonnet',
  tasks=[
    mvnTest(version)
    for version in versions
  ]
)
```

As you can see from [Tekton Dashboard](https://github.com/tektoncd/dashboard), with [Tekton CLI](https://github.com/tektoncd/cli), or just kubectl itself, the above Jsonnet script would generate 3 Pods to be executed in parallel. In each Pod, the steps are almost identical, with the Docker image versions changing. You can extend the tekton.jsonnet file to support features like timeouts, runAfter, result conditions. Jsonnet allows external parameters as strings or Javascript objects, this can be helpful to generate more dynamic pipelines.

However, you could also write the above by using `Task CRD` make it modular to get Java version and branch as a parameter, but the above Jsonnet code feels cleaner to me and is more extensible if you ever have a templating need arise. I repeat string templating YAML is an abomination. Jsonnet is not doing string interpolation, it's Turing-complete, and it 's much easier to catch errors, (but it also has its some difficulties). Beware, if you go multiple-step ahead, you can find yourself replicating Jenkins's Groovy menace and can cause tears. Maybe your pipelines should not be that complex, but if they ever are, Jsonnet+Tekton is a great combination you can try.