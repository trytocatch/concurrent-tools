#Concurrent tools

---
##1. BoundlessCyclicBarrier

中文介绍请看这：[BoundlessCyclicBarrier](http://www.cnblogs.com/trytocatch/p/boundlesscyclicbarrier.html)

A concurrent tool, like CyclicBarrier, it has a version, the thread will be blocked who calls await. For the `awaitWithAssignedVersion`, if the current version is equal to or later than assigned version, the caller thread won't be blocked, otherwise it will be blocked to wait the assigned version reached.

When calls `cancel`, the waiting threads will be unblocked. If calls `nextCycle` in quick succession, the threads which blocked doesn't with a assigned version is certain to be unblocked, but the threads blocked with assigned version may not be unblocked, it may still wait for the assigned version's reach. 

**Note:** the version can only increase, overflow is OK. 

This tool is thread-safe, beside all those wait methods, the others are nonblocking.

---
##2. LatestResultsProvider

中文介绍请看这：[LatestResultsProvider](http://www.cnblogs.com/trytocatch/p/latestresultsprovider.html)

This is a tool for working out the latest result.

It was written for this situation: **you want do some time-consuming calculations for some data and get the result, but the data changes frequently by multiple threads, and it's meaningless to continue with the calculation once the data has changed. So it need to cancel the current calculation with low cost and it must be thread-safe.**

The request has divided into two operation, 'update the parameters' and 'update', once you changed the data, you need to call the method `updateParametersVersion` to let this tool know, if there is a calculation on the way, it will be canceled(the task left to you is canceling your calculation in the implementation of calculateResult once you detect that `isWorking`  returns false) and restart, or do nothing in other cases. The `update` mean that you want to get the latest result, it will launch the calculation if the data has changed since the last calculation finished, otherwise nothing will be done. There are several variants of `update`, such as `updateAndWait`, you can use it to wait for the result. It will stop waiting if a newer result has worked out or the method `stopCurrentWorking` was called or that thread was interrupted. 

If the calculation is hard to cancel, and the data changes frequently, you can use `setUpdateDelay` to delay the launch of the calculation. But this will lead to another issue(or you may not think it is an issue), the data changes so frequently that the calculation never be launched because its interval is less than `UpdateDelay`. If you think it is an issue, you can use `setDelayUpperLimit` to avoid it. If the sum of all `UpdateDelay` greater than `DelayUpperLimit`, the delay mechanism will be disabled for this time. 

The methods except `updateAndWait` are nonblocking at **wait-free** level. 

As described in 'Package java.util.concurrent.atomic Description':

    compareAndSet and all other read-and-update operations such as getAndIncrement have the memory effects of both reading and writing volatile variables.

So there is no thread-safe problem if you call `updateParametersVersion` after changing the data, the new data will be visible at `calculateResult` even through there is no synchronization has been made.

`updateParametersVersion` happens-before the worker thread enter `calculateResult` in its turn. 

**How to use it:**

Implement the abstract method `calculateResult` to do your calculation with your data, cancel your calculation if you detect that the method `isWorking` returns false, call the method `updateParametersVersion` once the data has changed, call the method `update` or `updateAndWait` to launch the calculation if you want to get the result.

