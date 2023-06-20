## How does work flow? : comparing workflow engines: flyte, cadence
###     *--- notes of study in progress*

Uber/Lyft ride-sharing services support user sessions which are relatively long time compared to other web services (web search typically require sub-second result response). For example in UberEat, from the time when user place order, then restaurant prepare, to the time when driver pick up and finally deliver, this whole process can take tens of minutes. To handle this kind of user sessions in simple and fault tolerant software is a challenge, for which Uber/Lyft created their own workflow systems as solution. This is a initial study note comparing uber's workflow engine: cadence (later temporal) and lyft's workflow engine: flyte. It is a study in progress.

### 1. similar program design and abstractions:
1. in both flyte and cadence, applications are designed as:
   * workflow function (main logic/control flow), driving/calling
   * activities/tasks functions (performing specific work items)

2. workflows functions: should be deterministic, idempotent,  
    produce exactly the same result if executed multiple times  
   * avoid non-determinism caused by external world:  
       do not allow direct interactions with outside APIs,  
        * rpc call to same api endpoint with same parameters could fail, or return different results at different time
        * these operations are done/isolated in tasks/activities
   * avoid non-determinism caused by internal operations:  
        avoid language features which involve non-determinism,  
        replace them with runtime supported deterministic features:
        * flyte/python:
           * avoid non-deterministic operations, such as: rand.random, time.now(), etc.
           * use flyte provided "conditional" constructs instead of python native conditionals.
        * cadence/(Go,Java,...): use runtime api for following:
           * get current time: workflow.Now()
           * thread/goroutine sleep: workflow.Sleep()
           * do not iterate over map/HashMap, it is pseudo random
           * do not use native Go.goroutine, Java.threads which are preemptively scheduled:  
               use runtime provided api:  
               * Go (runtime coroutine):  
                    workflow.Go(), workflow.Channel, workflow.Selector
               * Java: Async.function, Async.procedure
                         
3. task/activity functions:
   * interact with external world, do real work:  
       call external apis, perform IO operations,..., which could fail
   * workflows invoke tasks/activities: passing arguments, return values  
        -- main data flows in worflows executions
   * main purpose of task/activity:  
     isolation of fallible logic/code into a independent unit (failure domain) which can be managed separately:
       * unit for performing timeout, retry, heartbeat-monitor and
       * history record:  
           whenever a task/activity function call start or complete, its invocation arguments and results will be recorded in event history db
          + flyte: optional, caching results can avoid duplicated tasks calls
          + cadence: fundamental, necessary for workflows fault tolerance

4. fault tolerance:
    * fault tolerance of application depends on:
      * deterministic, idempotent workflow functions  
          so it can be replayed to achieve same output
      * fallible code are isolated into tasks/activities,  
	  which can be guarded by timeout, retry, and heartbeating,  
          whose invocations (arguments/results) can be saved to execution event history DB and can be recovered or replayed later (without duplicated external calls/side-effects) to restore last known workflow execution state.
           
    * common settings for fault-tolerance:  
        especially for task/activity with external interactions
       * time out
       * retry
       * heartbeat
              
    * flyte/cadence use different strategies for workflow fault recovery:
       * flyte:
          * enable (recovery?) caching of nodes/tasks/workflows
          * rerun execution in recovery mode: ```flytectl create execution --recover execution_id ...```
             * recover all node/tasks completed and cached
             * resume from failed node
       * cadence:  
                to resume after crash or eviction, restart the workflow at an available node, and replay recorded execution event history to restore last known state.

### 2. different computational model

1. flyte: async task model:
    * workflow is set of async tasks connected through futures/promises into an execution graph/DAG;
    * workflow functions are not code to run at runtime,  
	but a DSL specification of tasks and runs at packaging/compile time;  
	it compiles to the execution graph/DAG of tasks, encoded in protobuf/FlyteIDL;
    * workflow execution graphs are executed by cluster runtime service (FlytePropeller)
    * workflow/tasks state transitions are recorded in database,  
      optionally caching tasks' results to avoid duplicated calls
    * recover workflow state (after crash) by running in recover mode, which will recover completed/cached nodes and resume at failed node.
    * most data passed via task-function input-arguments/output-results
             
2. cadence: durable/fault-oblivious computing:
    * workflow functions are real code and control-flow to run at runtime,  
	with a "durable" execution stack and env, maintained or rebuilt (after crash) through event sourcing (CQRS)
    * workflows run in application instances/containers, driving itself.
    * cadence will not save workflow state, only state-changing events in DB:
      * activity-calls events(input-arguments/output-results)
      * other events: coroutine scheduling, timer,...
    * recover workflow state (stack, env) through restart and replay of events saved in event history DB (event sourcing/CQRS)
    * good for workflows with heavy complex service state: deep-stack, threads,..  
        it is cheaper and can reach more identical state through replay? 

### 3. different execution model in cluster:

1. flyte:
    * custom container image creating tools:  
        use python decorator @task, @workflow to annotate task functions and workflow functions;  
	during project packaging/registration step, running "pyflyte package/register" will automatically generate docker image and workflow execution graph in protobuf directly from code.
    * k8s integrated:
        * workflow/tasks registered with FlyteAdmin in cluster
        * workflow executions are launched as k8s CRD
        * workflow executions are carried out by a k8s controller FlytePropeller, which launch task containers, tracking status
    * unit of deployment: tasks, workflows, launched as individual containers
    * flyte runtime (FlytePropeller) directly control how a workflow is executed in cluster, FlytePropeller will drive through the execution graph as specified in protobuf spec.  
      -- centralized execution model
    * use @task decorator to specify:
        * fault-tolerant settings: timeout, retry, caching,  
            -- used and monitored by FlytePropeller
        * required computing resources: cpu, mem, gpu,...  
            -- these used by FlytePropeller for scheduling through k8s (cluster?) api
    * a well integrated, automated, "turn-key" solution,  
        user can start writing/running their python applications right away without the need to learn container tools (docker build), cluster tools (kubectl), etc.
             
2. cadence:
    * agnostic of clustering, composable with different clustering
    * use any existing tools to build container images
    * unit of deployment: instances of applications(cadence workers),  
        which contain/register workflows and tasks
    * cadence service is not in control of executing workflow;  
        cadence just start the workflow in proper application/worker instances; workflow will drive itself.  
        -- distributed execution model
    * cadence is a toolkit with sole focus on durable-computing,  
        its workflow/activity configs only specify settings for fault tolerance: retry, timeout, heartbeat...  
    * users (or his CICD) responsible for scheduling / deploying application/worker instances:  
        use cluster native facilities, deploy application instances to proper nodes with required resources: cpu, mem, gpu, node-types,...
    * a very flexible toolkit, composable with different cluster/schedulers;  
        users use image building tool, cluster tools to build/deploy/start applications:  
        eg. for integration with k8s:  
            write Helm Charts with node affinity to launch applications (with workflow, tasks) at proper nodes with required resources
