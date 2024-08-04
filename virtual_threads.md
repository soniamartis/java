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

