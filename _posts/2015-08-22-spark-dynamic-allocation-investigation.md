---
layout: post
title: "Investigation of Dynamic Allocation in Spark"
date: 2015-08-22
author: "Jerry Shao"
tags:
    - spark
    - cloud
---

# Investigation of Dynamic Allocation in Spark #

Dynamic executor allocation is an important mechanism in Spark for better resource scheduling. Today I will give a deep investigation of how this mechanism works, what is the pros and cons of this mechanism and future works we could do on this area.

## Why Dynamic Allocation Matters ##

As we all know Spark is a distributed computation engine, it has several deploy modes like Standalone, Mesos and Yarn, which means Spark can run on these cluster managers. Spark core itself does not care about resource scheduling and management, it will ask the cluster manager to get resources it wanted. This is the legacy design of modern computation engine, like MRv2, separate job scheduling and resource management.

But the difference between Spark and MRv2 is:

>Each Spark task is a thread which resides in a process called "**executor**", normally executor is a long running process launched at the start of Spark application, and be killed after the application is finished.

>While in MRv2, each task resides in a process, its lifetime is task based, which means it will be killed when the task is finished.

Thinking of each process as a resource unit, in Spark it will hold the resources until the end of application, while in MRv2, each resource unit will be released at run-time.

If your workload is the only application running on the cluster, this resource pre-acquisition will get better performance, since it doesn't need to acquire the resources in the run-time. But in a real, in-production environment, normally you're not the monopolizer of the cluster, this pre-acquisiton will introduce some resource scheduling problems:

1. Under utilise of cluster resources. For example, if you start a spark-shell application  with some resources, but do not submit even one job for a long time, at this situation these resources are under-utilized, though occupied by Spark, it is not a intended behaviour.
2. Starvation of other applications. If your application occupies lots of resources without releasing, other application will be queued for resource acquisition.
3. Lack of elastic resource scaling ability. It is hard to estimate the amount of reasonable resources beforehand. For example, if you're running a iterative workload in which data will be inflated gradually through iteration, a pre-esitimated resource amount is not a good choice.

So according to problems mentioned above, instead of pre-acquisiton of resources, is it possible to acquire and release the resources in the runtime according to the load of current application?


The **Dynamic Executor Allocation** is introduced to handle such issues, it is firstly brought in Spark 1.2, through the evolution and refinement of codes, now this functionality is more mature and robust.

## To Enable Dynamic Allocation Mechanism ##

To enable this function, you have to configure **spark.dynamicAllocation.enabled** to **true**, also enable **spark.shuffle.service.enabled**, because now executors will by dynamically added or removed, so external shuffle service should be enabled for shuffle data transmission.

Similar to the configuration of MRv2 external shuffle service, user should add *spark-<version>-yarn-shuffle.jar* to the classpath of NodeManager, besides below configurations should be added to yarn-site.xml:

    <property>
      <name>yarn.nodemanager.aux-services</name>
      <value>spark_shuffle</value>
    </property>

    <property>
      <name>yarn.nodemanager.aux-services.spark_shuffle.class</name>
      <value>org.apache.spark.network.yarn.YarnShuffleService</value>
    </property>

>Note: Spark on Yarn configuration `--num-executors` is not worked when dynamic allocation is enabled, you don't need to specify this.

Besides, there has several configurations for you to fine-grained control the behavior of dynamic allocation, below is the configurations pasted from Spark official document.


Property Name  | Default   | Meaning |
:------------- | :-------: | :-------
 spark.dynamicAllocation.executorIdleTimeout |	60s | If dynamic allocation is enabled and an executor has been idle for more than this duration, the executor will be removed. For more detail, see this description.`
spark.dynamicAllocation.cachedExecutorIdleTimeout | 2 * executorIdleTimeout| If dynamic allocation is enabled and an executor which has cached data blocks has been idle for more than this duration, the executor will be removed. For more details, see this description.
spark.dynamicAllocation.initialExecutors | spark.dynamicAllocation.minExecutors | Initial number of executors to run if dynamic allocation is enabled.
spark.dynamicAllocation.maxExecutors | Integer.MAX_VALUE | Upper bound for the number of executors if dynamic allocation is enabled.
spark.dynamicAllocation.minExecutors | 0 | Lower bound for the number of executors if dynamic allocation is enabled.
spark.dynamicAllocation.schedulerBacklogTimeout | 1s | If dynamic allocation is enabled and there have been pending tasks backlogged for more than this duration, new executors will be requested. For more detail, see this description.
spark.dynamicAllocation.sustainedSchedulerBacklogTimeout | schedulerBacklogTimeout | Same as spark.dynamicAllocation.schedulerBacklogTimeout, but used only for subsequent executor requests. For more detail, see this description.


## Inside of Dynamic Allocation Mechanism ##

Compared to statically request all the resources at the start of application, dynamic allocation mechanism could request and remove the resources dynamically at run-time. So what's the policy to request and remove the resources?

First we will discover the request policy of dynamic allocation mechanism.

### Request Policy ###

To request resources from cluster manager, basically we have to figure out two problems:

1. How to measure the resources currently Spark requires?
2. When to request the resources from cluster manager?

#### Resource in Spark ####

In Spark a resource unit is **executor**, executor is combined with a bunch of CPU cores and memory. Each executor is a unique resource unit requested to cluster manager. For example, if we set `spark.executor.cores` to 10 and `spark.executor.memory` to `10g`, which means this resource unit is a combination of 10 cores and 10g memory to request to cluster manager.

While in the Spark execution layer, the smallest running unit is **Task**, so how to represent tasks as resource unit and be measured and controlled in Spark? Spark defines each task occupies `spark.task.cups` number of CPU cores, so we could simply calculate the number of tasks which could simultaneously run a executor:

    private val tasksPerExecutor =
      conf.getInt("spark.executor.cores", 1) / conf.getInt("spark.task.cpus", 1)

So now we simply connect the resource unit (executor) with execution unit (task), we could calculate the mount of resources we wanted through tasks.

For example if we have 100 tasks pending to run and each task occupies 1 core, also each executor has 10 cores, ideally we will need 10 executors to run all these tasks simultaneously. We treat this 10 executors as a current request to send to cluster manager.

So as a conclusion:

>Spark internally bridge the resource unit (executor) and execution unit (task), and calculate the number of required resources (executors) through tasks.

#### How to calculate the desired resources (executors) ####

We've already mentioned about the resource unit in Spark and how to calculate the resources through tasks. Now we have another question, how a get a **desired** resource number?

Spark `ExecutorAllocationManager` has a sophisticated algorithm to calculate the desired executor number.

1. Spark calculate the maximum number of executors it requires through pending and running tasks:

        private def maxNumExecutorsNeeded(): Int = {
        	val numRunningOrPendingTasks = listener.totalPendingTasks + listener.totalRunningTasks
        	(numRunningOrPendingTasks + tasksPerExecutor - 1) / tasksPerExecutor
        }

2. If current executor number is more than the expected number:

        // The target number exceeds the number we actually need, so stop adding new
        // executors and inform the cluster manager to cancel the extra pending requests
        val oldNumExecutorsTarget = numExecutorsTarget
        numExecutorsTarget = math.max(maxNeeded, minNumExecutors)
        numExecutorsToAdd = 1

        // If the new target has not changed, avoid sending a message to the cluster manager
        if (numExecutorsTarget < oldNumExecutorsTarget) {
          client.requestTotalExecutors(numExecutorsTarget, localityAwareTasks, hostToLocalTaskCount)
          logDebug(s"Lowering target number of executors to $numExecutorsTarget (previously " +
            s"$oldNumExecutorsTarget) because not all requested executors are actually needed")
        }
        numExecutorsTarget - oldNumExecutorsTarget

	If the current executor number is more than the desired number, Spark will notify the cluster manager to cancel pending requests, since they are unneeded. For those already allocated executors, they will be ramped down to a reasonable number later through timeout mechanism.

3. If current executor number cannot satisfy the desired number:

        val oldNumExecutorsTarget = numExecutorsTarget
        // There's no point in wasting time ramping up to the number of executors we already have, so
        // make sure our target is at least as much as our current allocation:
        numExecutorsTarget = math.max(numExecutorsTarget, executorIds.size)
        // Boost our target with the number to add for this round:
        numExecutorsTarget += numExecutorsToAdd
        // Ensure that our target doesn't exceed what we need at the present moment:
        numExecutorsTarget = math.min(numExecutorsTarget, maxNumExecutorsNeeded)
        // Ensure that our target fits within configured bounds:
        numExecutorsTarget = math.max(math.min(numExecutorsTarget, maxNumExecutors), minNumExecutors)

        val delta = numExecutorsTarget - oldNumExecutorsTarget

        // If our target has not changed, do not send a message
        // to the cluster manager and reset our exponential growth
        if (delta == 0) {
          numExecutorsToAdd = 1
          return 0
        }

        val addRequestAcknowledged = testing ||
          client.requestTotalExecutors(numExecutorsTarget, localityAwareTasks, hostToLocalTaskCount)
        if (addRequestAcknowledged) {
          val executorsString = "executor" + { if (delta > 1) "s" else "" }
          logInfo(s"Requesting $delta new $executorsString because tasks are backlogged" +
            s" (new desired total will be $numExecutorsTarget)")
          numExecutorsToAdd = if (delta == numExecutorsToAdd) {
            numExecutorsToAdd * 2
          } else {
            1
          }
          delta
        } else {
          logWarning(
            s"Unable to reach the cluster manager to request $numExecutorsTarget total executors!")
          0
        }

    There are two things should be noted:

    * The actual request is triggered when there have been pending tasks for `spark.dynamicAllocation.schedulerBacklogTimeout` seconds, and then triggered again every `spark.dynamicAllocation.sustainedSchedulerBacklogTimeout` seconds thereafter if the queue of pending tasks persists, which means resource allocation request will not be issued immediately, still need to wait the backlog time.
    * The number of executors requested in each round increases exponentially from the previous round. For instance, an application will add 1 executor in the first round, and then 2, 4, 8 and so on in the subsequent rounds.


#### When to request resources (executors) ####

Inside the `ExecutorAllocationManager`, a timer based scheduling thread will be start in each 100ms to calculate the number of resources and request to cluster manager, the code snippet is shown below:

    val scheduleTask = new Runnable() {
      override def run(): Unit = {
        try {
          schedule()
        } catch {
          case ct: ControlThrowable =>
            throw ct
          case t: Throwable =>
            logWarning(s"Uncaught exception in thread ${Thread.currentThread().getName}", t)
        }
      }
    }
    executor.scheduleAtFixedRate(scheduleTask, 0, intervalMillis, TimeUnit.MILLISECONDS)

and:

    private def schedule(): Unit = synchronized {
      val now = clock.getTimeMillis

      updateAndSyncNumExecutorsTarget(now)

      removeTimes.retain { case (executorId, expireTime) =>
        val expired = now >= expireTime
        if (expired) {
          initializing = false
          removeExecutor(executorId)
        }
        !expired
      }
    }

Till now, we simply introduce the mechanism of how and when to calculate and request the desired resources. We didn't introduce the details of how to count the number of pending tasks, running tasks and completed tasks, this is achieved by `SparkListener`, you could check the code of `ExecutorAllocationListener` inside the `ExecutorAllocationManager`.

## How YARN supports Dynamic Allocation ##

We've already introduced about how to calculate the desired resources (executor numbers), now we have to issue these resource requests to the cluster manager to allocate/deallocate the resources. Here we will introduce how YARN support resource allocation and deallocation.

In the YARN side, resource unit is **container**, container is a resource unit combined by a bunch of CPU cores, memory. Here in Spark we assume each executor is equal to a container, so the allocation of executors now becomes allocation of containers in YARN side.

### Communicate with Application Master ###

When desired executor number is calculated by `ExecutorAllocationManager`, we need to communicate with ApplicationMaster about the number of executors we required, then let ApplicationMaster to communicate with YARN resource manager to request the containers. Here I will simply show the overall process:

1. Call `SparkContext#requestTotalExecutors` to update the number of executors.
2. `SparkContext#requestTotalExecutors` will call `CoarseGrainedSchedulerBackend#requestTotalExecutors` to update the executors.
3. `CoarseGrainedSchedulerBackend#requestTotalExecutors` will call `YarnSchedulerBackend#doRequestTotalExecutors` to send the request to AM.
4. AM will get the current total number of executors required and pass it to `YarnAllocator#requestTotalExecutorsWithPreferredLocalities`.
5. `YarnAllocator` will further allocate or deallocate the containers based on the current number of running and pending containers.

### Container Allocation ###

When current container number is less than the desired number, we need to increase the number of container by requesting to ResourceManager, Spark on Yarn did this in `YarnAllocator#updateResourceRequests`:

	val numPendingAllocate = getNumPendingAllocate
    val missing = targetNumExecutors - numPendingAllocate - numExecutorsRunning

    if (missing > 0) {
      logInfo(s"Will request $missing executor containers, each with ${resource.getVirtualCores} " +
        s"cores and ${resource.getMemory} MB memory including $memoryOverhead MB overhead")

      val containerLocalityPreferences = containerPlacementStrategy.localityOfRequestedContainers(
        missing, numLocalityAwareTasks, hostToLocalTaskCounts, allocatedHostToContainersMap)

      for (locality <- containerLocalityPreferences) {
        val request = createContainerRequest(resource, locality.nodes, locality.racks)
        amClient.addContainerRequest(request)
        val nodes = request.getNodes
        val hostStr = if (nodes == null || nodes.isEmpty) "Any" else nodes.last
        logInfo(s"Container request (host: $hostStr, capability: $resource)")
      }
    }

Here `missing` is the actual container number we wanted, if `missing > 0` which means current number of containers is not enough to satisfy the need, more containers need to be allocated. we will calculate the locality of containers and send the container requests to resource manager through:

    amClient.addContainerRequest(request)

The details of container locality calculation algorithm can be found here in `LocalityPreferredContainerPlacementStrategy`.

### Container Deallocation ###

When current container number is more than the desired number, we need to decrease the number of containers by cancelling the requests, here in Spark on Yarn, we did this by cancelling the pending container requests and killing the running containers explicitly.

**Cancel the pending container requests**

    val numPendingAllocate = getNumPendingAllocate
    val missing = targetNumExecutors - numPendingAllocate - numExecutorsRunning

    if (missing > 0) {
		...
    } else if (missing < 0) {
      val numToCancel = math.min(numPendingAllocate, -missing)
      logInfo(s"Canceling requests for $numToCancel executor containers")

      val matchingRequests = amClient.getMatchingRequests(RM_REQUEST_PRIORITY, ANY_HOST, resource)
      if (!matchingRequests.isEmpty) {
        matchingRequests.head.take(numToCancel).foreach(amClient.removeContainerRequest)
      } else {
        logWarning("Expected to find pending requests, but found none.")
      }
    }

** Kill the running containers **

When current container is idle for a while, which means there's no tasks running on the executors for a period of time, `ExecutorAllocationManager` will explicitly ramp down the number of containers by killing them.

The code in `ExecutorAllocationManager` can be seen in:

    private def schedule(): Unit = synchronized {
      val now = clock.getTimeMillis

      updateAndSyncNumExecutorsTarget(now)

      removeTimes.retain { case (executorId, expireTime) =>
        val expired = now >= expireTime
        if (expired) {
          initializing = false
          removeExecutor(executorId)
        }
        !expired
      }
    }

This killing request will be transmitted to Spark application master, when application master receives this message, it will explicitly kill the executors by calling `killExecutor`.

    /**
     * Request that the ResourceManager release the container running the specified executor.
     */
    def killExecutor(executorId: String): Unit = synchronized {
      if (executorIdToContainer.contains(executorId)) {
        val container = executorIdToContainer.remove(executorId).get
        containerIdToExecutorId.remove(container.getId)
        internalReleaseContainer(container)
        numExecutorsRunning -= 1
      } else {
        logWarning(s"Attempted to kill unknown executor $executorId!")
      }
    }

Internally it will kill the containers by AMRMClient:

    private def internalReleaseContainer(container: Container): Unit = {
      releasedContainers.add(container.getId())
      amClient.releaseAssignedContainer(container.getId())
    }

## Conclusion ##

Until now we simply investigate the mechanism of dynamic executor allocation. Currently dynamic executor allocation is no only for Yarn, but also can run in Standalone and Mesos mode. more features will be added into this mechanism to make it more robust. For better resource utilization and multi-tenancy sharing, dynamic executor allocation is an important feature for everyone to try.
