# Akka
Set of open source libraries that allows you to design scalable, resilient systems that span multi cores and networks. 
- Responsive
- Resilient
- Elastic
- Message Driven - asynchrounous

**motivation:**

	-  SBSS, how to avoid back pressure, right way to design distrbuted systems, how to handle synchronous tasks in an asynchronous api, running out of threads!?
	-  writing concurrent code is hard

- Akka relies on the actor model to handle parallel processing in a high performance network. Why do we need a better system than oop for writing concurrent code?<br/>
	
	***limitations with OOP***
	
	- We need to rely on locks for any sort of interleaving between threads. Locks limit concurrency and are very costly for the modern cpus as they require os interrupts to suspend and resume threads.
	- Further, when a caller thread is blocked and waiting on a lock it can't do any meaningful work. Its not even an option for UI applications.
	- Mostly importantly it introduces dead locks which are hard to debug
		- [eventual consistency](https://github.com/harsha-konda/Cloud-Computing/tree/master/Project3.3%7CMulti-threading%20Programming%20and%20Consistency/Eventual%20Consistency)
		-  [strong consistency](https://github.com/harsha-konda/Cloud-Computing/tree/master/Project3.3%7CMulti-threading%20Programming%20and%20Consistency/Strong%20Consistency)  
	-  no win situation: without locks -> state gets corrupted **vs** with locks -> performance suffers
	-  Objects can only guarantee encapsulation (protection of invariants) in the face of single-threaded access, multi-thread execution almost always leads to corrupted internal state. Every invariant can be violated by having two contending threads in the same code segment.<br/>
  	
  	***illussion of shared memory***
  	
	- There is no real shared memory anymore, CPU cores pass chunks of data (cache lines) explicitly to each other just as computers on a network do. Inter-CPU communication and network communication have more in common
	- Instead of hiding the message passing aspect through variables marked as shared or using atomic data structures, a more disciplined and principled approach is to keep state local to a concurrent entity and propagate data or events between concurrent entities explicitly via messages.

***real world use cases***

- https://stackoverflow.com/questions/4493001/good-use-case-for-akka

