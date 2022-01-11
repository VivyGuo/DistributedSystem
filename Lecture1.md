- What is distributed system ?
    
    A set of cooperating computers  are communicating with each other over network to get some coherent task done
    
    For example: storage for big websites, big data computation such as MapReduce, peer-to-peer file sharing 
    
- Why people build distributed system?
    
    <aside>
    💡 Before I even talk about distributed system, sort of remind you :
    
    If you're designing a system if you can possibly solve it on a single computer without distributed system you should do it that way, it's always easier. You should try everything else before you try building distributed system.
    
    </aside>
    
    Get high-performance: 
    
    - **parallelism** ⭐
        
        lots of CPUs, memories lots of disk run in parallel
        
    - **tolerate faults** ⭐
        
         two computers do the exact same thing if one of them fails you can cut over to the other one another
        
    - **physical**
        
        some computers are physically spread out,  for exapmle interbank transfers of money , bank A‘s computer is in New York City and Bank B's computer is in London
        
    - **security/isolated**
        
        if you need to interact with somebody , but there's some code you don't trust(maybe their code has bugs in it) , you want to split  the computation to other computer. you talk to each other to narrowly defined network protocol
        
- What make distributed system so hard?
    - concurrency
        
        there are multiple computers execute concurrently, which you get problems, complex interactions, timing-dependent 
        
    - partial faults
        - multiple computers **plus a network**
        - some computers stopped working, other computers continue working or the computers are working but some part of the **network is broken or unreliable**
    - performance
        
        it's tricky to obtain that thousand X speed up with a thousand computers
        

Topics of the Course

- Infrastructure
    - Storage
        
        it **turns out** that storage is going to be the one we focus most on because it's a very well-defined and useful abstraction and usually fairly straightforward abstraction
        
    - Communication
    - Computation
        
        
- Abstraction
    
    it's rarely fully achieved but the dream would be to be able to build  interfaces that look like a non distributed  system
    
- Implementation
    - RPC (remote procedure call)
        
        abstraction hidden the network underneath
        
        (mask the fact that we're communicating over an unreliable Network)
        
    - threads
        
        harness multi-core computers and **structure concurrent operations** 
        
    - concurrency control
        
        lock
        
- Performance
    
    scalability or scalable speed-up
    
    if you increase  X computers, you get X times throughput
    
- **Fault Tolerance - Method⭐**
    - Availability
    - Recoverability
        
        like hard drives or flash or solid-state drives 
        
    - Method
        - non-volatile storage (非易失性存储，持久性: 断电后恢复)
            - hard drives or flash or solid-state drives
            - store a checkpoint or a log of the state of a system and then when the power comes back up  we'll be able to read our latest state and continue from there
            - avoid writing NV storage too much because non-volatile storage tends to be expensive to update
            - writing NV storage meant was moving a disk arm and waiting for a disk platter to rotate, both of which are agonizingly slow on the scale of  three gigahertz microprocessors
        - replication
            
            we have two servers each with a supposedly identical copy of the system state
            
            the two replicas will accidentally drift out of sync and will stop being replicas right
            
- Consistency ⭐
    - Why are there inconsistency?
        
        **multiple copies** or **caching**: there are lots of different versions of data ,replica servers are hard to keep identical. 
        
    - Strong guarantee
        
        Get(k) yields the value from the **most recent Put(k,v)** 
        
        require expensive communication: 
        
        - update all copies
        - read all copies, use the latest timestamp data
    - Weak guarantee
        
        ofen choose weaker systems for performance
        
        only update and consult **the nearest copy**
