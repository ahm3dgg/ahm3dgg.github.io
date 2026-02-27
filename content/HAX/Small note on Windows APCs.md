APCs are short for asynchronous procedure calls, the motivation behind them was to allow executing code in a context of a target thread and though in its process address space from another thread.

That was the motivation, however as with every engineering problem, comes more problems, these APCs are asynchronous, they can run at arbitrary points, example they can execute in the middle of updating some shared state, or they can execute while the thread is holding a lock, suppose for instance that the target thread was holding a spinlock and we queued an APC, what will happen if that APC attempted to acquire that spinlock ? a dead lock will occur ! nothing will save you from that, not even preemption because remember its the same thread, its as if you acquired a lock twice for example like this

```cpp
acquire_spinlock(&splock)
acquire_spinlock(&splock)
```

even if the lock is recursive, actually this will break your code also, you will be modifying shared state from two independent execution paths.

so how can we fix this ? the solution is to only execute APCs when we are allowed to, which Microsoft terms as `Alertable` state, when a thread enters this Alertable state, we can start executing the queued APCs, so executing at Alertable state wasn't the goal but a workaround for the problem !

A Thread can enter an Alertable state for example when calling `NtDelayExecution` 

```cpp
NTSYSCALLAPI 
NTAPI 
NtDelayExecution( _In_ BOOLEAN Alertable, _In_ PLARGE_INTEGER DelayInterval );
```

Specifying `Alertable` with `TRUE` will allow user-mode APCs to be executed.

Other examples are `NtWaitForSingleObject`, `NtWaitForMultipleObjects` , these points are safe because the thread is waiting here its not doing anything.

So for user-mode APCs they are only executed when the thread is in `Alertable` state or there is an explicit call for a function like `NtTestAlert`, which can 
be used to flush the APC queue doing so will execute all APCs the currently held in the Thread's APC queue, if you ever heard about Early Bird APC Injection, this function makes this technique possible simply because ntdll calls it just before calling the main entry point of the loaded module.

for normal kernel-mode APCs, if a normal kernel-mode APC is executing further normal kernel-mode APCs are disabled until that APC finishes its execution. You may also want to read about special APCs.

Since this is a small note, and obviously I didn't cover all the details, 
I will just leave you with some uses of APCs in the kernel, but before just as a disclaimer I did reference ReactOS and WRK here, so things might have changed in later windows versions, however the use cases still show how APCs can be useful so maybe you can try to think about them and why APCs was actually a nice solution to the problem.

```
- NtSuspendThread (APC That waits on a Semaphore)

- NtSetContext/NtGetContext (APC That captures thread context)

- IopfCompleteRequest (APC That copies IoStatus to user buffer)

- NtTerminateThread (APC That lets the Thread Terminate it self)

- LdrInitializeThunk also runs from an APC:
  
  -> NtCreateThread 
	  -> PspCreateThread 
		  -> KeInitThread (with PspUserThreadStartup) 
			  -> PspUserThreadStartup 
				  -> KiInitializeUserApc (with SYSTEM_DLL.LoaderInitRoutine = LdrInitializeThunk)
```