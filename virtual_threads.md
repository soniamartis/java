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
- Option 2 means u cannot use imperative code anymore, and will have to switch to smaller atomic operations

