# Virtual Threads

- Virtual threads are super-lightweight. They occupy negligible amount of memory.
- We can literally create millions of virtual threads in the application compared to only thousands of regular threads in a real application
- Java now has 2 types of threads, carrier threads that are mapped to the operating system and virtula threads that live inside the jvm
- The virtual thread is mounted on the carrier thread
- The task runs on the virtual thread, and when it blocks, the virtual thread gets blocked, however, the virtual thread is unmounted from the carrier thread
- The carrier thread can now take some other request and start working on them
- When the I/O completes, the task + virtual thread get mounted back onto the carrier thread
- This way we will need much fewer carrier threads

## Demo
```java
 public static void doSomething(int index){
        System.out.println(index + " thread entering " + Thread.currentThread());
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        System.out.println(index + " thread leaving " + Thread.currentThread());
    }

    public static void main(String[] args) {
        System.out.println("Hello world!");

        try(var executorService = Executors.newVirtualThreadPerTaskExecutor()){
            for(int i=0;i<10;i++){
                var idx = i;
                executorService.submit(() -> doSomething(idx));
            }
        }

    }
```
Simplified Output snippets
```
2 thread entering VirtualThread[#22]/runnable@ForkJoinPool-1-worker-3
2 thread leaving VirtualThread[#22]/runnable@ForkJoinPool-1-worker-4
3 thread entering VirtualThread[#23]/runnable@ForkJoinPool-1-worker-4
3 thread leaving VirtualThread[#23]/runnable@ForkJoinPool-1-worker-3
```

Observation
- The virtual thread is bound to the task as can be seen from index and virtual thread name
- The carrier thread is no longer bound to the task, so we see one carrier thread enter the method, and another leave the method
- Had we used the former thread implementation, we would see the same carrier thread bound to the task

```java

try(var executorService = Executors.newFixedThreadPool(5)){
            for(int i=0;i<10;i++){
                var idx = i;
                executorService.submit(() -> doSomething(idx));
            }
        }
```

Output
```
1 thread entering Thread[#20,pool-1-thread-2,5,main]
1 thread leaving Thread[#20,pool-1-thread-2,5,main]
4 thread entering Thread[#23,pool-1-thread-5,5,main]
4 thread leaving Thread[#23,pool-1-thread-5,5,main]
```

---
## Analysing CPU usage of blocking code
- Consider this simple code snppet that makes a request to get user by name and then deseriliaze it to user object
- Overall this code completes in about 100 ns
- CPU busy time for this request = 0.0001%
```java
Json request = buildUserRequest(name);  --> in-memory compute;CPU busy, say 10ns
String userJson = userServer.userByName(request); --> CPU is idle, waiting for response from network, 100ms
User user = Json.unmarshal(userJson); --> in-memory compute;CPU busy 10ns
```
- The most common model that comes to mind to handle mutiple such requests is the 1 thread per request model
- But, can we still keep our CPU busy 100% of the time with this model
- How many threads are required to keep CPU busy 100% of the time

| # Threads | % CPU usage |
|------|------|
| 1| 0.0001%|
| 10 | 0.001%|
|100| 0.01%|
|1000|0.1%|
|10,000| 1%|
|100,000|10%|
|1,000,000|100%|

- From above math, it takes a milltion threads to keep the CPU busy 100% of the time
- A java thread is a thin wrapper over the OS kernel thread
- This OS thread consumes quite some resources
- Typically, creating a thread requires 2MB of memory upfront, so 1 million threads will require 2TB of memory
- Creating a thread also takes time, in the order of a millisecond
- Creating a million threads may take 1000 seconds which is close to 15 mins
- If we try to create thse threads on our system ,we will hit a limit long before that
- This means that the thread per request is not a great model to handle large number of concurrent requests

- At this point there are 2 options to achieve higher requests per second
- Either use a different types of threads that are lightweight compared to kernel threads or launch multiple requests per platform thread
- Option 2 means u cannot use imperative code anymore, and will have to switch to smaller atomic operations written in lambdas and wire them up together in the async framework
- Imperative vs Reactive programming example of user saving their items in shopping acrt and payment with success email

 ```java 
User user = userService.findUserByName(name);
if(!repo.contains(user)){
  respo.save(user);
}
var cart = cartService.loadCartFor(user);
var total = cart.items().stream().mapToInt(Item::price).sum();
var txnId = paymentService.pay(user,total);
emailService.send(user,cart,txnId);
```
- This code is easy to read, write, maintain, write tests, debug
- Adding new logic to this code is also very simple, as it is a piece of simple, imperative, step-by-step code. It is almost boring
- But this will not scale
- Async code scales well, but is hard to read, maintain, add new logic, debug and test
- If something goes wrong with the imperative code, it is easy to trace the issue as everything is present in the stacktrace
- But in case of async frameworks, it is the framework that handles the inputs and outputs to the lambdas, so they dont appear in the stacktrace
- Exception handling, handling timeouts is hard with async
- So cost of maintenance with async is super high
- Since async is hard, we go for option 1 ie. use ligtweight threads that are 1000 times less expensive than platform threads
- This way, we dont need to write async code anymore
- Virtual threads are lighter than platform threads by a factor of more than a thousand
- They are so lightweight, u dont need to pool them anymore
- Concurrency concepts for virtual threads stay the sme as platform threads, ie visibility, atomiciy, transient, synchronized, locks etc
- These virtual threads on the jvm's forkjoinpool which is a pool of platform threads
- Th VT is mounted on the platform thread in this pool, and the task is executed by the platform thread through the VT
```java
Runnable task = () -> {
 User user = userService.findUserByName(name);
 if(!repo.contains(user)){ // VT detects that it is performing a blocking I/O. it unmounts from the platform thread and moves it's context to heap memory
   respo.save(user);
 }
 var cart = cartService.loadCartFor(user);
 var total = cart.items().stream().mapToInt(Item::price).sum();
 var txnId = paymentService.pay(user,total);
 emailService.send(user,cart,txnId);
}
```
- When VT detects that there is a blocking call, it calls continuation.yield() that copies the stack of the VT to the heap and unmounts it from platform thread
- Once the data from the blocking call is available, the OS handler that monitors this data will trigger a signal to continuation.run
- The continuation will then get the stack of the VT back from heap and puts it in the waitlist of the platform thread it was mounted on in the FJP
- If the original PT is busy and another thread within the FJP is available, PT2 will steal the task from the first PT

### Cost and Caveats of VTs
- The cost of running a task in VT is higher than the cost of running the task on the platform thread, but the cost of blocking a VT is tremendously low than blocking a PT
- Use VT only if u have tasks that do a lot of blocking computations, dont use it for in-memory computations, its useless
- Dont use parallelStreams with VTs, and dont use parallelStreams for I/O operations
- Garbage collector and JIT compiler run in platform threads as they are in-memory computations
- Caveats are VT thread pinning to PT
- A VT is pinned to PT in 2 scenarios
  - while executing native code (call to C/C++ code)
  - in a synchronized block
- A performance issue can occur if a VT is pinned to PT for too long in the below scenario:
- What can we do if we are in a situation where the VT is pinned to PT due to synchronized block:
  - Analyse the logic within the sync block/native code, if it is simply in-memory logic like guarding a in-memory datastucture, then its probably ok, as this operation could take a few nanos
  - If the synch code is doing a blocking operation in synch block, then we may have to replace it with a ReentrantLock, so that the VT doesnt get pinned to PT

```java
Lock lock = new ReentrantLock();
try{
  lock.lock();
}finally{
  lock.unlock();
}
}
```

## Final takeaways
- new kind of threads in Java space
- Cheap to create, you can have million of them, cheap to blck
- You do not need to rely on non-blocking async code anymore
- Running some blocking code in VT is fine, pls do it
- You do not need to pool them, create them on demand, use them and let them die at the end, thats how the jvm uses them too
- You should typically run only blocking I/O code in VT, not in-memory code, it is useless
- When u run native code/ synchronized block, the VT gets pinned to PT, which may be an issue if u are doign I/O in sync block
