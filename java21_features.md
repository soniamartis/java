# Java 21 new features

### Regular expression use of named groups
```java
String line = "1;New York;8 336 817";
Pattern pattern = Pattern.compile("""
              (?<index>\\d+);\
              (?<city>[ a-zA-Z]+);\
              (?<population>[ \\d]+)$
""");

Matcher matcher = pattern.matcher(pattern);
if(matcher.matches(line)){
  var index = matcher.group("index");
  var city = matcher.group("city");
  var population = matcher.group("population");
}


```

### SequencedCollection, SequencedSet and SequencedMap
- watch episode 19 of jEP cafe

### Addition of creation methods to collections API
```java
var map = HashMap.newHashMap(100);
var set = HashSet.newHashSet(100);
var linkedMap = LinkedHashMap.newLinkedHashMap(100);
var linkedSet = LinkedHashSet.newLinkedHashSet(100);
```

### Autocloseable for several java classes
- HttpClient
- ExecutorService
- ForkJoinPool
- Autocloseable is a very nifty method that helps u close a resource once it is not needed
- It has a single close() method
- We can use a try-with resources only if that class implements AutoCloseable
- When we reach the end of this try-with block, the resource will close no matter what
```java
class MyConnection implements AutoCloseable {
   void close(){
     // close the resources that are no longer needed
   }
}

try(var conn = new MyConnection()){
  // do something with connection
}
```

### Thread API changes
- The Thread class sleep and join now support methods that take in a Duration object
- New methods to create threads
- New method in exec service to create V threads, these threads are created on the fly and not pooled
```java
void sleep(Duration duration);
void join(Duration duration);
boolean isVirtual();

Thread.ofPlatform();
Thread.ofVirtual();
var executor = Executors.newVirtualThreadPerTaskExecutor();
```

### Deprecated APIs
- finalize() method
- new constructors on wrapper classes like new Integer(num)-> Integer.valueOf(num) in preparation of value types from the valhalla project