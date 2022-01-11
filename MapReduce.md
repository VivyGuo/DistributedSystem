# Introduction

## What is MapReduce?

- Answer
    - a programming model : process and generate large data sets
        
        (automatically parallelized and executed  on a large cluster of commodity machines. )
        
    - map function that **processes** a key/value pair <k1,v1> to **generate** a set of intermediate key/value pairs <k2, v2>
    - reduce function that **merges** all intermediate values associated with the same intermediate key <k2, v3>

## The Run-time System of MapReduce

- Answer
    1. partition the input data
    2. **schedule** the program's **execution** across a set of machines
    3. **handle** machine failures
    4. manage the required inter-machine communication

# Programming Model

## Strucutre

- Map
    - written by user
    - read an input pair <k1,v1>  from GFS (local disk)
    - produce a set of intermediate key/value pairs.  list(k2,v2)  **(R files, one per reduce task)**
    - write intermediate to local disk
- Library (coordinator)
    - gather all intermediate values associated with the same intermediate key I
    - pass them to the Reduce function via an iterator(network). <k2, list(v2)> a set of v2
- Reduce
    - written by user
    - read an intermediate key I and a set of values for that key from Map output (network)
    - produce  output <k2,v3>  **(one output file)**
    - write output to GFS (global  file system)
    

## Example

**Count of Word**

Consider the problem of **counting the number** of **occurrences of each word** in a large **collection of documents**. The user would write code similar to the following pseudo-code:

```
  map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
				EmitIntermediate(w, "1");

  reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
      result += ParseInt(v);
	    Emit(AsString(result));
```

- The map function
    
    emits **each word** plus **an associated count of occurrences** (just ‘1’ in this simple example).
    
- The reduce function
    
    sums together **all counts** emitted for a particular word.
    

**Count of URL Access Frequency**

- The map function
    
    **processes** logs of web page requests and **outputs** ⟨URL, 1⟩. 
    
- The reduce function
    
    **adds** together all **values for the same URL** and **emits** a ⟨URL, total count⟩ pair.
    

**Reverse Web-Link Graph**

- The map function
    
    **outputs** ⟨target,source⟩ pairs for **each link(每个链接) to(指向) a target URL(目标URL)** found in a page named source. (为指向→在页面中找到的→目标URL的→每个链接)
    
- The reduce function
    
    concatenates (链接) the list of all source URLs(源URL) associated with a given target URL and emits the pair:⟨target, list(source)⟩  (链接→与给定目标URL→关联的→所有源URL的→列表)
    

More examples : search index, sort

# Implementation

## MapReduce Execution Overview

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ae4b3d17-9642-464a-850b-64a397dd2ec9/截屏2021-08-11_下午12.26.20.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ae4b3d17-9642-464a-850b-64a397dd2ec9/截屏2021-08-11_下午12.26.20.png)

### There are

- a master process
    
    hands out tasks to workers and copies with the failed workers
    
- worker processes
    
    call map and reduce function and handle reading and writing files
    

<aside>
💡 Question?????

1. What's the content of worker, and what's the content of tasks?
    
    How does the worker handle  tasks?
    
2.  Does reduce worker start until all map workers complete?
    
    I guess yes
    

We subdivide the map phase into M pieces and the reduce phase into R pieces.

Having each worker perform many different tasks improves dynamic load balancing, and also speeds up recovery when a worker fails: the many map tasks it has completed can be spread out across all the other worker machines.

</aside>

### Task quantity

We subdivide the **map phase** into M pieces and the **reduce phase** into R pieces

- In practice
    
    **M** : make sure that each individual task is roughly 16 MB to 64 MB of input data 
    
    **R** :  a small multiple of the number of **worker machines**
    
    like  M = 200, 000 and R = 5, 000, using 2,000 **worker machines**
    

**Map Library**

- split input data into 64MB files
- start up master, master  **assign** map or reduce tasks to worker

### Map worker read

- read input data from local disk (via GFS)
- parse key/value pairs from input data
- pass key/value pairs to map function

**map function generate**

- generate intermediate key/value pairs
- buffer intermediate data in memory

### Map worker write

- buffered pairs are written to local disk **periodically**, and local disk is partioned into R regions by partitioning function, hash(intermediate key); R files**(one temporary file per reduce task)**
- **send** complete state of map task and **location** and size of R files to master

### Reduce worker read

master notify reduce worker and send location of R files to reduce worker

- reduce worker read intermediate data from the local disks of the map workers via **RPC**
- reduce worker **sort** the intermediate data by keys so that all occurrences of **the same key are grouped** together.
- reduce workder pass the key and a set of intermediate values to reduce function

**reduce function**

- produce output

### Reduce worker write

- output of reduce function appened to a **final** output file for this reduce **partition**
- users  do not need to combine the final output files into one file, because they can pass the files as input to another MapReduce call, or use them frmo another distributed application
- 💡  Each in-progress task writes its output to private **temporary** files
    - one map task produce **R temporary files** (one per reduce task)
    - one reduce task produce **one temporary file**, when a reduce task completed, reduce worker atomically r**enames its temporary outfile file to final output file**

![截屏2021-08-17 上午1.20.50.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7892e9ab-a028-409a-9718-ac6575f14b23/截屏2021-08-17_上午1.20.50.png)

When all map tasks and reduce tasks have been completed, the master wakes up the user program. At this point, the MapReduce call in the user program returns back to the user code.

---

- 1.  MapReduce library
    - **splits the input files** into M pieces of typically 16 megabytes to 64 megabytes (MB) per piece (controllable by the user via an optional parameter).
    - **fork**:  starts up many copies of the program on a cluster of machines.
- 2. Master assign worker
    
    The master picks idle workers and **assigns** each one a map  or a reduce task.
    
- 3. Map worker read,buffer,generate
    - **Map worker : read, parse, pass**
        
        1) **read**:  A worker who is assigned a map task **reads** the contents of the corresponding input split. (GFS, local disk)
        
        2) **parse**: It parses key/value pairs out of the input data 
        
        3) **passes** each pair to the **user-defined *Map* function.(负责produce)** 
        
    - Map function : **produce**, **buffer in memory**
        
               **The intermediate key/value pairs** produced by the *Map* function are buffered in memory.
        
- 4. Map worker write, partition, location
    - write: **Periodically**, the buffered pairs are **written to local disk**, **partitioned into R regions**  by the partitioning function.
    - location: The **locations** of these buffered pairs on the local disk **are passed back to the maste**r, who is responsible for **forwarding(send) these locations to the reduce workers**.
- 5. Reduce worker read,rpc, sort
    - read:  When a reduce worker **is notified by the master** about these locations, it uses **remote procedure calls(RPC)** to read the buffered data from the local disks of the map workers. (network)
    - sort: When a reduce worker has **read all intermediate data**, it **sorts it by the intermediate keys** so that all occurrences of the same key are **grouped** together. The sorting is needed because typically many different keys map to the same reduce task.
        
         If the amount of intermediate data is too large to fit in memory, **an external sort is used.(外部排序)**
        
- 6.Reduce worker write,  not combine
    - The reduce worker **iterates** over(遍历) the sorted intermediate data and for each unique intermediate key encountered, it **passes** the **key** and the corresponding **set of intermediate values** to the user’s ***Reduce* function.(负责produce)**
    - The **output** of the *Reduce* function is **appended to** a final output file for this reduce partition.
    - Not  combine R output files into one file
        
        After successful completion, the output of the MapReduce execution is available in the R output files (one per reduce task, with file names as specified by the user).
        
        Typically, users **do not need to combine these R output files into one file** – they often pass these files as input to another MapReduce call, or use them from another distributed application that is able to deal with input that is partitioned into multiple files.
        

---

## Master Data Structures

- state of task
    
    for each map task and reduce task, the master stores the state (idle, in-progress, completed)
    
- identity of  worker
    
    the identity of the worker **machine** (for non-idle tasks)
    
- location and size of the intermediate file
    
    for each completed map task, the master stores th**e locations and sizes** of the R intermediate file region(R份中间文件区域). 
    
    The master moves the locations and sizes from map tasks to reduce tasks
    

## Fault Tolerance

- When is the worker failed state?
    
    The master **ping** every worker periodically. If no response is received from a worker in a certain amount of time, the master **marks** the worker as failed.
    
- When is the worker idle state?
    
    Map tasks **completed** by the worker are reset  to  idle state.
    
    Map tasks or reduce tasks in progress on a **failed** worker are  reset to idle
    

### Worker Failure

- What map and reduce task do when failure ?
    - c**ompleted** map task and **not-completed** reduce task
        
        **Completed m**ap task  are **re-executed**  because their output is stored on the **local** disk(s) of the failed machine and is therefore inaccessible. 
        
        When a map task is executed first by worker A and then later executed by worker B (because A failed), all workers executing reduce tasks are notified of the re-execution. Any reduce task that has not already read the data from worker A **will read the data from worker B**
        
    - **completed** map and c**ompleted** reduce task
        
        both **not** need to be re-executed since reduce task already fetch the intermediate data and output is stored in a **global** file system.
        

### Master Failure

- What about master failure?
    - Multiple master
        
        Make the master write periodic(定期写入) checkpoints of the master data structures . 
        
        **A new copy** can be started from the last checkpointed state
        
    - Single master
        
        However, given that there is only a single master, its failure is unlikely(不太可能); 
        
        Therefore our current implementation **aborts** the MapReduce computation.
        
        Clients can check for this condition and retry the MapReduce operation if they desire.
        

### Semantics in the Rresence of Failure

(故障出现时的语义)

- Why map and reduce  must be deterministic function ?
    - Deterministic function make sure that  **distributed implementation** produces the same output as would have been produced by a **non-faulting sequential execution** of the entire program.(即使故障时，分布式**并发产生**的输出和无故障顺序执行产生的输出一致，即每次map结果都一致,都是同一reduce)
    - Suppose a map task runs twice, one reduce task sees first run's output, another reduce task sees the second run's output.
- What if  non-functional map or reduce ?
    
    Worker failure would require whole job to be re-executed.
    
- What is pure deterministic function?
    - The function is allowed to look at their arguments. no state, no file I/O, no interaction, no external communication.
    - atomic commit of map task  and atomically rename of reduce task to achieve this property
- What is atomic commit of map task?
    - When a map task completes, the worker sends a message to the master and includes the names of the R temporary files in the message.
    - If the master receives a **completion message** **for an already completed** map task, it **ignores the message**.
    - Otherwise, it records the names of R files in a master data structure.
- What is atomically rename of reduce task?
    - When a reduce task completes, the reduce worker **atomically renames** (provided by the **underlying file system**) its temporary output file to the final output file.
    - If the same reduce task is executed on multiple machines, multiple rename calls will be executed for **the same final output file.**

## Locality/Task Granularity/Backup tasks

- What is locality?
    - Input data (managed by GFS) is stored the local disks of the cluster machines
    - The map workers **read input data locally(from GFS)** and consumes no network bandwidth.
    - GFS divides each file into 64 MB blocks, and stores several copies of each block (typically 3 copies) on different machines.
- What is task granularity(粒度)?
    
    subdivide the map phase into M pieces and the reduce phase into R pieces
    
    - Ideally
        
        M and R should be much larger than the number of worker machines.
        
        - improves **dynamic load balancing**
        - **speeds up recovery** when a worker fails
    - in practice
        - M  make sure that each map task is roughly 16 MB to 64 MB of input data
        - R  is a small **multiple** of the number of **worker** machines (小倍数)
        
        M = 200, 000 and R = 5, 000, using 2,000 worker machines.
        
- What is backup task?
    - To alleviate the problem of stragglers: a machine that takes an unusually long time to complete one of the last few map or reduce tasks.
    - When a MapReduce **operation is close to completion**, the master **schedules backup executions** of the remaining in-progress **tasks**.
    - The task is marked as completed whenever either the primary or the backup execution completes.
    - the mechanism typically increases the computational resources  **by no more than** a few percent. (不超过)
- Why we need backup task?
    
    Redundant execution can be used to reduce the impact of **slow machines**, and to handle **machine failures and data loss**.
    

# Refinements

### 1. Partitioning Function

- What is partitioning function?
    
    used on the **intermediate** key specify the **number** **of reduce** tasks/output files 
    
- Default partitioning function？
    - hashing (e.g. “hash(key) mod R”)
    - fairly well-balanced partitions
- Special partitioning function？
    - hash(Hostname(urlkey)) mod R
    - When the keys are URLs, this function causes **all URLs from the same host** are in the **same output file.**

### 2. Ordering Guarantees

- Why we need ordering for the intermediate key?
    - efficient random access **lookups** by key
    - convenient to **find** data in sorted output

### 3. Combiner Function

- Why we need combiner function?
    - There is **significant repetition** in the intermediate keys and will be sent over the network to a single reduce task
    - **To minimize network use(save network bandwidth),** the combiner function that does **partial merging** of this data before it is sent over the network.
- Where is the combiner function executed on ?
    
    E**ach machine that performs a map task**.
    
- What's difference between a reduce function and a combiner function ?
    
    It is how the MapReduce library handles the output of the function. 
    
    - The output of a reduce function is written to the final output file.
    - The output of a combiner function is written to an intermediate file that will be sent to a reduce task.

### 4. Input and Output Types

- What is function of a reader interface?
    - support for reading input data or produce output data in several different formats
    - reads records from a database, or from data structures mapped in memory.

### 5. Skipping Bad Records

- Why we need to skip bad records?
    - Sometimes there are bugs in user code that is **not feasible**; perhaps the bug is in a **third-party library** for which source code is unavailable.
    - Sometimes bugs is **acceptable** to ignore a few records;for example  statistical analysis
- How does MapReduce skip bad records?
    - The MapReduce library **stores the sequence numbe**r of the argument in a global variable. Each **worker** process **installs** **a signal handler** that catches segmentation violations and bus errors.
    - If the **user code generates a signal,** **the signal handler** **sends** a “last gasp” UDP packet that contains **the sequence number** **to the  master.**
    - When the master has seen more than one failure on a particular record, it indicates that the record should be skipped when it issues the **next re-execution** of the corresponding Map or Reduce task.(当下一次重新执行相应task 发生时)

### 6. Local Execution

- Why we need local execution?
    - **Debugging problems  in a distributed systemcan be tricky**, since the actual computation happens often on several thousand machines,with work assignment decisions made dynamically by the master.
    - To help facilitate debugging, profiling, and small-scale testing, we have developed an alternative implementation of the MapReduce library that **sequentially executes all of the work** on the local machine.
- What is local execution?
    - Controls are provided to the user so that the computation can be limited to particular map tasks.
    - Users invoke their program with a special flag and can then easily use any debugging or testing tools they find useful.

### 7. Status Information

- The master runs an internal HTTP server and exports a set of status pages for human consumption. The status pages show :
    - **the progress of the computation**, such as how many tasks have been completed, how many are in progress,
    - **bytes of input**, bytes of intermediate data, bytes of output, processing rates
    - the standard **error** and standard output files
    - which workers have failed, and which map and reduce tasks they were processing when they failed.

### 8. Counters

The MapReduce library provides a counter facility to **count occurrences of various events**. For example, count total number of words processed 

- How to use counter?
    
    User code creates a named **counter object** and then increments the counter appropriately in the Map and/or Reduce function.
    
    ```
    Counter* uppercase;
    uppercase = GetCounter("uppercase");
    map(String name, String contents):
        for each word w in contents:
    			if (IsCapitalized(w)):
    				uppercase->Increment();
    			EmitIntermediate(w, "1");
    ```
    
    - The counter values from **individual worker machines** are periodically propagated to the master . **The master aggregates the counter values**  and returns them to the user code when the MapReduce operation is completed.
    - The master **eliminates  duplicate** executions of the same map or reduce task to **avoid double counting.**

# Performance

- T**wo computations**  m**easure the performance** of MapReduce
    - **search** **data** : extracts a small amount of interesting data from a large data set.
    - **sort** **data**: shuffles data from one representation to another
- Why we need the network file system?
    
    We want  flexibility  to read any piece of input on any worker server, so network file system store the input data 
    
- What will likely limit the performance?
    - In 2004 MapReduce were limited by **network capacity.**
        
        In MR's all-to-all shuffle, half of traffic goes through root switch.  Paper's root switch: 100 to 200 gigabits/second, total 1800 machines, so 55 megabits/second/machine. 55 is small, e.g. much less than disk or RAM speed.
        
        ![截屏2021-08-17 上午12.10.23.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/80e7ebc7-779e-45fd-b170-91eca1582639/截屏2021-08-17_上午12.10.23.png)
        
    - Today: networks and root switches are much faster relative to CPU/disk network
        
        the modern data center networks have **far more network throughput**,  the modern MapReduce actually no longer tried to run the maps on the same machine before Google stopped using the MapReduce
        
        ![截屏2021-08-17 上午12.16.10.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ee014171-816c-4022-9d6e-65caacd7e71a/截屏2021-08-17_上午12.16.10.png)
        

### **How does MR minimize network use?** ⭐

- How does MR minimize network use?
    
    (target at reducing the amount of data sent across the network)
    
    - locality
        
        **GFS and map workers** run on the **same** machine, the **map and reduce** worker run **different** machine : 
        
        - map worker **read** data from l**ocal disks** via GFS , not over network.
        - map worker **write**  the intermediate data to **local disk** (GFS)
        - reduce worker read intermediate data **via network**
        - reduce worker write output to GFS
    - partioned,sort, merge intermediate data
        - Intermediate data **partitioned** into R files, R is much smaller than the number of keys.
        - reduce worker **sorts** Intermediate data  by the intermediate keys
        - combiner function that does **partial merging** of this data
- How does MR get good load balance?
    - many more tasks than workers
        
        M and R  are much larger than the number of worker machines
        
    - redundant execution of task
        
         reduce the impact of **slow machines**, and to handle **machine failures and data loss**
        
        - When a MapReduce **operation is close to completion**, the master **schedules backup executions** of the remaining in-progress **tasks**.
        - The task is marked as completed whenever either the primary or the backup execution completes.
- What's the expensive part of the MapReduce ?
    - the transformation from row storage to column storage, the paper calls a **shuffle**
        - originally the keys is **stored by row** on the same machine that run  map worker
        - then the keys is **stored by column** on the machine that run reduce worker
            - Graph 1
                
                ![截屏2021-08-17 上午1.20.50.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1287aeb1-ce50-43a1-9634-59be86e8ed9f/截屏2021-08-17_上午1.20.50.png)
                
    - amount of data across the network
        
        moving the intermediate data **across the network** from the map that produced it to the reduce that would need it , it's the expensive part
        
        - output data of map  can be as large as output of  reduce for sorting
        - output data of reduce can be smaller  than output of map for merging

# On the Death of Map-Reduce at Google

现在，我们更接近于更全面地兑现软件承诺。随着我们[对集成分布式和本地执行模型的方式](http://research.microsoft.com/en-us/um/people/jrzhou/pub/scope-vldbj.pdf)有了[更好的理解，](http://research.microsoft.com/en-us/um/people/jrzhou/pub/scope-vldbj.pdf) MPP 数据库概念远非与大型无共享部署完全不兼容，而是变得越来越适用。我记得坐在剑桥微软研究院的一个阅读小组里，我们讨论了连接在集群计算中是否有效率。答案是肯定的，而且当时的技术已经为人所知。交易同样被认为属于“永远无法工作”的阵营，但[Spanner 已经表明在该领域取得了进展](http://research.google.com/archive/spanner.html). 也许 OLTP 永远不会批发到集群计算，但这些集群中的数据不需要是只读的。

随着这些更通用的框架的改进，它们包含了 Map-Reduce 并使其缺点更加明显。Map-Reduce 从来都不是一个容易编写新程序的范式，因为你的问题和刚性两相拓扑之间的映射很少很明显。语言只能在一定程度上掩盖这种阻抗不匹配。Map-Reduce 在实现时通常会产生大量开销，这归因于其固有的“批量性”，以及需要在 map 和 reduce 阶段之间设置障碍。为最终用户提供更好的选择是一种解脱

Now we are much closer to delivering much more fully on the software promise. MPP database concepts, far from being completely incompatible with large shared-nothing deployments, are becoming more and more applicable as we develop a [better understanding of the way to integrate distributed and local execution models](http://research.microsoft.com/en-us/um/people/jrzhou/pub/scope-vldbj.pdf). I remember sitting in a reading group at Microsoft Research in Cambridge as we discussed whether joins could ever be efficient in cluster computing. The answer has turned out to be yes, and the techniques were already known at the time. Transactions are similarly thought to be in the ‘can never work’ camp, but [Spanner has shown that there’s progress to be made in that area](http://research.google.com/archive/spanner.html). Perhaps OLTP will never move wholesale to cluster computing, but data in those clusters need not be read-only.

As these more general frameworks improve, they subsume Map-Reduce and make its shortcomings more evident. Map-Reduce has never been an easy paradigm to write new programs for, if only because the mapping between your problem and the rigid two-phase topology is rarely obvious. Languages can only mask that impedance mismatch to a certain extent. Map-Reduce, as implemented, typically has substantial overhead attributable both to its inherent ‘batchness’, and the need to have a barrier between the map and reduce phases. It’s a relief to offer end-users a better alternative

So it’s no surprise to hear that Google have retired Map-Reduce. It will also be no surprise to me when, eventually, Hadoop does the same, and the elephant is finally given its dotage.
