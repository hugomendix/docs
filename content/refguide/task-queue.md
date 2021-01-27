---
title: "Task Queue"
parent: "resources"
menu_order: 85
description: "Concepts and usage of the task queue"
tags: ["task queue", "process queue", "parallel", "scheduling", "microflow"]
---

## 1 Introduction

Using a **Task Queue** allows you to run microflows asynchronously while controlling the number of microflows that are executed simultaneously by assigning them to a task queue. You can configure the task queue to control the maximum load put on your application by these microflows during peak usage while still ensuring all microflows are eventually executed.

### 1.1 Replacing the Process Queue module

This way of executing tasks in the background supersedes the earlier [Process Queue](https://docs.mendix.com/appstore/modules/process-queue) App Store module.  
See the section [Replacing Process Queue](#process-queue), below, for more information on the differences between the two mechanisms.

## 2 Configuration

Microflows can be scheduled to run in the background when they are initiated using a **Call Microflow** action in Studio Pro, or through the Java API.

In a single node scenario, these tasks will simply be executed on the single node.

In a clustered setting, the Mendix runtime distributes these tasks transparently throughout the cluster. Should a cluster node be shutdown or fail halfway during executing a task, then the remaining cluster nodes will pick it up (eventually, when the node is detected to be down) and re-execute it. This happens automatically and does not need to be managed.

### 2.1 Creating Task Queues

Background execution is done in so called **Task Queues**. They can be created in Studio Pro as follows:

1. Right click on a module or folder.
2. Select **Add other**.
3. Click **Task Queue**.
4. Enter the value for **Threads** for each cluster node (maximum 40).

    Task Queues have a number threads. Each of these threads can process one task at a time. That is, a queue will pick up as many concurrent tasks as it has threads. Whenever a task is finished, the next one will be picked up.
    
    In general, one or two threads should be enough, unless there is a large number of tasks or tasks take a long time and need to execute in parallel. Having many threads will put additional load on the database and should not be done if not needed.

### 2.2 Scheduling Microflow Executions

#### 2.2.1 In Studio Pro

In Studio Pro, a [Call Microflow](microflow-call) activity can start a microflow in a Task Queue.

1. Edit the **Call Microflow** activity.
2. Check the box **Execute this Microflow in a Task Queue**.
3. Set **Select Task Queue** to the task queue in which the microflow should be executed.

For microflows which are running in a task queue, the context in which the microflow runs changes slightly in the following ways:

 * Microflows are always executed in a *sudo* context, even when the scheduling microflow has **Apply entity access** set to *true* (see [Microflow Properties](microflow) for more information).
 * Only committed persistable entities can be passed as parameters to the microflow. Passing a persistable *New* or *Changed* entity produces a runtime error. Basically, this means an entity must have been committed previously or is committed in the same transaction in which the task is created.
 - The microflow is not executed immediately. The task is added only to a task queue when (and if) the transaction from which it has been scheduled ends successfully. At that point any cluster node may pick it up.
 - If the execution fails with an exception, the failure is logged in the `System.ProcessedQueueTask` table.

#### 2.2.2 Through the API

The `Core` class in `com.mendix.core` has been extended with a method [microflowCall](https://apidocs.rnd.mendix.com/8/runtime/com/mendix/core/Core.html#microflowCall(java.lang.String)). It can be used to schedule a microflow for background execution as in the following example:

```java
Core.microflowCall("AModule.SomeMicroflow")
  .withParam("Param1", "Value1")
  .withParam("Param2", "Value2")
  .executeInBackground(context, "AModule.SomeQueueName");
```

The method `executeInBackground` takes two parameters: a context and a queue name. The context is only used for creating the queue; executing the task will be done with a system context.

Scheduling a microflow to be executed returns immediately. The microflow will be executed somewhere in the cluster, as soon as possible after the transaction in which it was called completes. Because the microflow is executed in the background there is no return value. The microflow will be executed in a *sudo* context.

{{% alert type="info" %}}
The context in which a background task runs is still under discussion and may change in the future.
{{% /alert %}}

### 2.3 Configuration options{#configuration}

The period for a graceful shutdown of queues can be configured.

| Configuration option          | Example value | Explanation                                              |
|-------------------------------|---------------|----------------------------------------------------------|
| `TaskQueue.ShutdownGracePeriod` |          10000| Time in ms to wait for task to finish when shutting down.|

The total number of worker threads is limited to 40 (per cluster node). There is no hard limit on cluster nodes.

### 2.4 Interfacing the queue

Besides scheduling and executing tasks, the Mendix platform keeps track of tasks that have been executed in the background: for example, which completed and which failed.

Internally, a scheduled or running task is represented by the Mendix entity `System.QueuedTask`. In a high performance setting, this entity should *not* be used directly by user code, because the underlying database table is heavily used. For example counting how many `System.QueuedTask` objects exist at the moment will lock the table and might cause a serious slowdown in task processing. You should also not Write directly to `System.QueuedTask`. Instead, mark a task for background execution in the **Call Microflow** activity or using the Java API.

Tasks that have been processed, that is have completed or failed, are saved as objects of entity type `System.ProcessedQueueTask`. These objects are at the user's disposal. They might be used, for example, to do the following:

1. Reschedule failed tasks if desired (this should be done by creating new task(s)),
2. Verify that tasks have run successfully, or
3. Debug the application in case of errors.

`System.ProcessedQueueTasks` objects are never deleted. The user is free to delete them when desired.

### 2.5 Task status

The **Status** attribute of `System.QueuedTask` and `System.ProcessedQueueTask` reflects the state that a background task is in. The values are:

* `Idle`: The task was created and is waiting to be executed.
* `Running`: The task is being executed.
* `Completed`: The task executed successfully.  A `System.ProcessedQueueTask` is added to reflect this.
* `Failed`: The task is no longer executing, because an exception occurred. A `System.ProcessedQueueTask` containing the exception is added to reflect the failure. The task will not be retried.
* `Aborted`: The task is no longer executing, because the cluster node that was executing it went down. A `System.ProcessedQueueTask` is added to reflect this. The task will be retried on another cluster node.
* `Incompatible`: The task never executed, because the model changed in such a way that it cannot be executed anymore. This could be because the microflow was removed/renamed, the arguments were changed or the Task Queue was removed.

### 2.6 Model changes

During the startup of the Mendix runtime, there is a check to ensure that scheduled tasks in the database fit the current model. The following conditions are checked:

* that the microflows exist
* that the parameters match
* that the queue exists 

If any of these condition checks fail, tasks are moved to `System.ProcessedQueueTasks` with **Status** `Incompatible`. The Runtime will only start after all scheduled tasks have been checked. This should in general not take very long, even if there are thousands of tasks.

### 2.7 Shutdown

During shutdown, the `TaskQueueExecutors` will stop accepting new tasks. Running tasks are allowed a [grace period](#configuration) to finish. After this period, the runtime will send an interrupt to all task threads that are still running and again allow a grace period for them to finish. After the second grace period the runtime just continues shutting down, eventually aborting the execution of the tasks. The aborted tasks will be reset, so that they are re-executed later or on another cluster node.

{{% alert type="info" %}}
Interrupting task threads may cause them to fail. 
{{% /alert %}}

## 3 Monitoring

To monitor tasks in the Task Queue the `QueueHelpers` module can be used in Mendix 9.

Since this hasn't been released yet, it can be downloaded through [Gitlab](https://gitlab.rnd.mendix.com/runtime/taskqueuehelpers-module) by Mendix employees only.

### 3.1 Logging

A Log Node exists specifically for all actions related to Task Queue, named `TaskQueue`.

## 4 Other

Executing **Find usages** on a task queue only finds the occurrences of that queue in microflows.

{{% alert type="info" %}}
Invocations from Java actions are not found.
{{% /alert %}}

### 4.1 Limitations

Task queues have the following limitations:

* Microflows that are executed in the background execute as soon as possible in the order they were created, but possibly in parallel. They are consumed in FIFO order, but then executed in parallel in case of multiple threads. There is no way to execute only a single microflow at any point in time (i.e. ensure tasks are run sequentially), unless the number of threads is set to 1 and there's only a single runtime node.
* Microflows that are executed in the background can *only* use the following types of parameters: Boolean, Integer/Long, Decimal, String, Date and time, Enumeration, committed Persistent Entity.
* Microflows that are executed in the background use a sudo/system context with all permissions. It is not possible to use a user context with limited permissions.
* Background microflows will start execution as soon as the transaction in which they are created is completed. This ensures that any data that is needed by the background microflow is committed as well. It is not possible to start a background microflow immediately, halfway during a transaction. Note that if the transaction is rolled back, the task is not executed at all.
* The total amount of parallelism per node is limited to 40. This means that at most 40 queues with parallelism 1 can be defined, or a single queue with parallelism 40, or somewhere in between, as long as the total does not exceed 40.
* Queued actions that have failed can't be rescheduled out-of-the-box currently. You can set up a scheduled microflow to re-attempt failed tasks. They can be queried from `System.ProcessedQueueTask` table.

### 4.2 High level implementation overview

Tasks are stored in the database in a `System.QueuedTask` table. For each background task a new object is inserted with a `Sequence` number, `Status = Idle`,  `QueueName`, `QueueId`, `MicroflowName`, and `Arguments` of the task. This happens as part of the transaction which calls the microflow and places it in the task queue, which means that the task will not be visible in the database until that transaction completes successfully.

The tasks are then consumed by executors that perform a `SELECT FOR UPDATE SKIP LOCKS` SQL statement, that will try to claim the next free task. The `SKIP LOCKS` clause will skip over any tasks that are already locked for update by other executors. The corresponding `UPDATE` changes the `Status` to `Running` and sets the owner of the task in the `XASId` and `ThreadId` attributes.

After the task has been executed, it is moved to be an object of the `System.ProcessedQueueTask` entity with `Status` `Completed` or `Failed`. If the task failed with an exception, this is included in the `ErrorMessage` attribute.

Arguments are stored in the `Arguments` attribute as JSON values. Arguments can be any primitive type ([variable](variable-activities))or a committed persistent object, which is included in the `Arguments` field by its Mendix identifier. Upon execution of the task, the corresponding object is retrieved from the database using the Mendix identifier. For this reason the persistent object must be committed before the task executes, because otherwise a runtime exception will occur.

When a node crashes, this is eventually detected by another cluster node, because it no longer updates its heartbeat timestamp. At this point the other node will reset all tasks that were running on the crashed node. The reset performs the following actions:

* create a copy of the task as a `System.ProcessedQueueTask` object with `Status = Aborted`
* set the `Status` back to `Idle`
* increment the `Retried` field
* clear the `XASId` and `ThreadId` fields

The task will then automatically be consumed again by one of the remaining nodes in the cluster. Effectively, this means that a task is guaranteed to be executed at least once.

{{% alert type="warning" %}}
Under normal circumstances, a task is executed exactly once, but in the face of node failures a task may be (partially) executed multiple times. This is the best guarantee that a distributed system can provide.
{{% /alert %}}

### 4.3 Replacing Process Queue{#process-queue}

The **Task Queue** supersedes the earlier [Process Queue](https://docs.mendix.com/appstore/modules/process-queue) App Store module, which has been deprecated with the release of Mendix 9. There are several differences between the Process Queue module and the **Task Queue**:

* The **Task Queue** supports a multi-node cluster setup and can therefore be used in a horizontally scaled environment.
* The **Task Queue** does not require additional entities to be created, since Microflows can simply be marked to execute in the background.
* The **Task Queue** does not yet support automatic retrying of failed tasks.
