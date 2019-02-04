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
	- OOP only enforces encapsulation. Encapsulation dictates that the internal data of an object is not accessible directly from the outside; it can only be modified by invoking a set of curated methods. The object is responsible for exposing safe operations that protect the invariant nature of its encapsulated data. This all gets violated when we run multi threaded code.
	
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

https://doc.akka.io/docs/akka/2.5/general/terminology.html

***real world use cases***

- https://stackoverflow.com/questions/4493001/good-use-case-for-akka


When is it a bad use case:
- You don't allow concurrency
- You don't allow mutable state (as in functional programming)
- You must rely on some synchronous communications mechanism


**Actor Hierarchy: **

- Use of Akka relieves you from creating the infrastructure for an actor system and from writing the low-level code necessary to control basic behavior. 
- An actor in Akka always belongs to a parent. Typically, you create an actor by calling context.actorOf()
![hierarchy](https://doc.akka.io/docs/akka/2.5/guide/diagrams/actor_top_tree.png)

	- the first instance of an actor is created is using `system.actorOf()`
	- `/` the so-called root guardian. This is the parent of all actors in the system, and the last one to stop when the system itself is terminated.
	- `/user` the guardian. This is the parent actor for all user created actors. Don’t let the name user confuse you, it has nothing to do with end users, nor with user handling. Every actor you create using the Akka library will have the constant path /user/ prepended to it.
	- `/system` the system guardian.

	
**creating actors**		

``` scala
  val system = ActorSystem("testSystem")

  val firstRef = system.actorOf(PrintMyActorRefActor.props, "first-actor")
  println(s"First: $firstRef")
  firstRef ! "printit"
```

```
First: Actor[akka://testSystem/user/first-actor#1053618476]
Second: Actor[akka://testSystem/user/first-actor/second-actor#-1544706041]

```

** basic actor api**

- `self` reference to the ActorRef of the actor
- `sender` reference sender Actor of the last received message, typically used as described in Actor.Reply
- `supervisorStrategy` user overridable definition the strategy to use for supervising child actors
- context exposes contextual information for the actor and the current message, such as:
 - factory methods to create child actors (actorOf)
 - system that the actor belongs to
 - parent supervisor
 - supervised children
 - lifecycle monitoring
 - hotswap behavior stack 

**sending messages **
- Messages are sent to an Actor through one of the following methods.

- `!` means “fire-and-forget”, e.g. send a message asynchronously and return immediately. Also known as tell.
- `?` sends a message asynchronously and returns a representing a possible reply. Also known as ask.
- Message ordering is guaranteed on a per-sender basis

```
victim ! Kill
```
**Receive messages**

```scala
class MyActor extends Actor {
  val log = Logging(context.system, this)

  def receive = {
    case "test" ⇒ log.info("received test")
    case _      ⇒ log.info("received unknown message")
  }
}
```

**Actor LifeCycle:**

- Actors pop into existence when created, then later, at user requests, they are stopped. Whenever an actor is stopped, all of its children are recursively stopped too.
- `context.stop(self)`
- Akka api exposes many lifecycle hooks
	- preStart() is invoked after the actor has started but before it processes its first message.
	- postStop() is invoked just before the actor stops. No messages are processed after this point.
	- postRestart() is invoked when an actor is restarted by parent. default bahviour of this mathod is to invoke preStart()
	- preRestart() is invoked just before restart. default behavious is to kill all children

![life-cycle](https://doc.akka.io/docs/akka/2.5/images/actor_lifecycle.png)
	

**Fault Tolerance**

- Each actor is the supervisor of its children, and as such each actor defines fault handling supervisor strategy.


```scala

import akka.actor.OneForOneStrategy
import akka.actor.SupervisorStrategy._
import scala.concurrent.duration._

override val supervisorStrategy =
  OneForOneStrategy(maxNrOfRetries = 10, withinTimeRange = 1 minute) {
    case _: ArithmeticException      ⇒ Resume
    case _: NullPointerException     ⇒ Restart
    case _: IllegalArgumentException ⇒ Stop
    case _: Exception                ⇒ Escalate
  }
  
```
OneForOne => each child is handle on an independent case by case basis based on the exception

OneForMany => suggestive


- DeathWatch

``` context
import akka.actor.{ Actor, Props, Terminated }

class WatchActor extends Actor {
  val child = context.actorOf(Props.empty, "child")
  context.watch(child) // <-- this is the only call needed for registration
  var lastSender = context.system.deadLetters

  def receive = {
    case "kill" ⇒
      context.stop(child); lastSender = sender()
    case Terminated(`child`) ⇒ lastSender ! "finished"
  }
}
```

https://github.com/harsha-konda/MapReduce-Akka


**Messages and immutability**

- Messages can be any kind of object but have to be immutable. 
- Scala can’t enforce immutability (yet) so this has to be by convention. 
- Primitives like String, Int, Boolean are always immutable. Apart from these the recommended approach is to use Scala case classes which are immutable (if you don’t explicitly expose the state) and works great with pattern matching at the receiver side.


**Stashing Msg**

```
import akka.actor.Stash
class ActorWithProtocol extends Actor with Stash {
  def receive = {
    case "open" ⇒
      unstashAll()
      context.become({
        case "write" ⇒ // do writing...
        case "close" ⇒
          unstashAll()
          context.unbecome()
        case msg ⇒ stash()
      }, discardOld = false) // stack on top instead of replacing
    case msg ⇒ stash()
  }
}
Java
Invo
```

Whats Next:

- become ?
- scheduler
- pararalle messages
- mailbox
- why should you to stash
- real world example
- amazon example - mailbox
- unit testing



- Dispatchers
- Mailboxes : An Akka Mailbox holds the messages that are destined for an Actor
- Routing
- FSM
- Persistance
- clustering - https://doc.akka.io/docs/akka/2.5/common/cluster.html
- streams - https://doc.akka.io/docs/akka/2.5/stream/index.html


- fault tolreant implementation 

- futures: https://doc.akka.io/docs/akka/2.5/futures.html#introduction
- Akka patterns: https://doc.akka.io/docs/akka/2.5/howto.html

**Extending Actors using PartialFunction chaining**

- Sometimes it can be useful to share common behavior among a few actors, or compose one actor’s behavior from multiple smaller functions.
-  This is possible because an actor’s receive method returns an Actor.Receive, which is a type alias for PartialFunction[Any,Unit], and partial functions can be chained together using the PartialFunction#orElse method. 

# 2019-02-01
**Interaction Patterns**

1. Fire and Forget: 
	 
	 **! -> tell**
	
	```scala
	case class PrintMe(message: String)
	
	val printerBehavior: Behavior[PrintMe] = Behaviors.receive {
	  case (context, PrintMe(message)) ⇒
	    context.log.info(message)
	    Behaviors.same
	}
	
	val system = ActorSystem(printerBehavior, "fire-and-forget-sample")
	
	// note how the system is also the top level actor ref
	val printer: ActorRef[PrintMe] = system
	
	// these are all fire and forget
	printer ! PrintMe("message 1")
	printer ! PrintMe("not message 2")
	
	```
	
	- Pros:	
		- Not critical to ensure if the message has been processed
		- We want to minimize the number of messages created to get higher throughput 
	- Cons:
		- If the inflow of messages is higher than the actor can process the inbox will fill up and can cause `OutOfMemoryError `
		- If the message gets lost, the sender will not know

2. Request Response :

	**? -> ask**	
	The ask pattern involves actors as well as futures. It is not offered as a default method on `ActorRef`
	
	```scala
	import akka.pattern.{ ask, pipe }
	import system.dispatcher // The ExecutionContext that will be used
	final case class Result(x: Int, s: String, d: Double)
	case object Request
	
	implicit val timeout = Timeout(5 seconds) 
	val future: Future[Result] = myActor ? "hello"
	
	//val future = myActor.ask("hello")(5 seconds) alternate way to 
	// express the above

	val f: Future[Result] =
	  for {
	    x ← ask(actorA, Request).mapTo[Int] // call pattern directly
	    s ← (actorB ask Request).mapTo[String] // call by implicit conversion
	    d ← (actorC ? Request).mapTo[Double] // call by symbolic name
	  } yield Result(x, s, d)
	
	// future is a monad
	future onSuccess {
	  case "bar"     => println("Got my bar alright!")
  	  case x: String => println("Got some random string: " + x)
	}
		
    // pipe it instead of waiting a
    pipe(future) to self


	future onComplete {
  	  case Success(result)  => doSomethingOnSuccess(result)
  	  case Failure(failure) => doSomethingOnFailure(failure)
	}
	```

***Digression: WTF is a monad?***	
https://www.coursera.org/lecture/progfun2/lecture-1-4-monads-98tNE

1. A monad is parametric type `M[T]` with two operations: `flatMap` and `Unit` 

	```
	trait M[T]{
	  def flatMap[T](f: T => M[U]): M[U]
	}
	
	def unit[T](x: T):M[T]
	```

2. map as a FlatMap and Unit
	
	```
	m map f == m flatMap(x => unit(f(x)))
	```
3. For an operation to be a monod it should obey associativity, left unit law, right unit law

	*why should i care about this?*
	
	allows us to refactor a series of flatMaps and Maps into a for expression
	
	
**Become/UnBecome**

Akka supports hotswapping the Actor’s message loop at runtime: invoke the context.become method from within the Actor. 
	
```scala
class HotSwapActor extends Actor {
  import context._
  def angry: Receive = {
    case "foo" ⇒ sender() ! "I am already angry?"
    case "bar" ⇒ become(happy)
  }

  def happy: Receive = {
    case "bar" ⇒ sender() ! "I am already happy :-)"
    case "foo" ⇒ become(angry)
  }

  def receive = {
    case "foo" ⇒ become(angry)
    case "bar" ⇒ become(happy)
  }
}
```

https://github.com/akka/akka-samples/blob/2.5/akka-sample-fsm-scala/src/main/scala/sample/fsm/DiningHakkersOnFsm.scala
http://www.dalnefre.com/wp/2010/08/dining-philosophers-in-humus/

**Stash/Unstash**
- stashes all the messages at a point
```scala
import akka.actor.Stash
class ActorWithProtocol extends Actor with Stash {
  def receive = {
    case "open" ⇒
      unstashAll()
      context.become({
        case "write" ⇒ // do writing...
        case "close" ⇒
          unstashAll()
          context.unbecome()
        case msg ⇒ stash()
      }, discardOld = false) // stack on top instead of replacing
    case msg ⇒ stash()
  }
}
```

**Dispatchers**
- An Akka MessageDispatcher is what makes Akka Actors “tick”, it is the engine of the machine so to speak. All MessageDispatcher implementations are also an `ExecutionContext`, which means that they can be used to execute arbitrary code, for instance Futures. 
- They can also be used to configure message guarentees at least once, at most once, exactly once.

```
implicit val executionContext = system.dispatchers.lookup("my-dispatcher")

my-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "fork-join-executor"
  //https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html
  //https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html
  # Configuration for the fork join pool
  fork-join-executor {
    # Min number of threads to cap factor-based parallelism number to
    parallelism-min = 2
    # Parallelism (threads) ... ceil(available processors * factor)
    parallelism-factor = 2.0
    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}

```

**MailBoxes**

An Akka Mailbox holds the messages that are destined for an Actor. Normally each Actor has its own mailbox, but with for example a BalancingPool all routees will share a single mailbox instance.

```
bounded-mailbox {
  mailbox-type = "akka.dispatch.NonBlockingBoundedMailbox"
  mailbox-capacity = 1000 
}
akka.actor.mailbox.requirements {
  "akka.dispatch.BoundedMessageQueueSemantics" = bounded-mailbox
}

```

**Interaction Patterns**
https://github.com/sksamuel/akka-patterns


1. Grouping Actor
	
	```
	import akka.actor.{Terminated, Actor, ActorRef}
	import scala.collection.mutable.ListBuffer

	class GroupingActor(count: Int, target: ActorRef) extends Actor {

	  val received = new ListBuffer[AnyRef]
	
	  override def receive = {
	    case Terminated(targ) => context.stop(self)
	    case msg: AnyRef =>
	      received.append(msg)
	      if (received.size == count) {
	        target ! received.toList
	        received.clear()
	      }
	  }
	}
	```
2. Priority Mailbox
	```
	import akka.dispatch.{MessageQueue, MailboxType}
	import akka.actor.{ActorSystem, ActorRef}
	import com.sksamuel.akka.patterns.{Envelope, PriorityAttribute}
	import java.util.{Comparator, PriorityQueue}
	import akka.dispatch
	import com.typesafe.config.Config
	
	class PriorityMailbox(settings: ActorSystem.Settings, config: Config) extends MailboxType {
	
	  def create(owner: Option[ActorRef], system: Option[ActorSystem]): MessageQueue = new PriorityMessageQueue
	
	  class PriorityMessageQueue
	    extends PriorityQueue[dispatch.Envelope](11, new EnvelopePriorityComparator)
	    with MessageQueue {
	
	    def cleanUp(owner: ActorRef, deadLetters: MessageQueue): Unit = {
	      if (hasMessages) {
	        var envelope = dequeue()
	        while (envelope ne null) {
	          deadLetters.enqueue(owner, envelope)
	          envelope = dequeue()
	        }
	      }
	    }
	    def hasMessages: Boolean = size > 0
	    def numberOfMessages: Int = size
	    def dequeue(): dispatch.Envelope = poll()
	    def enqueue(receiver: ActorRef, e: dispatch.Envelope): Unit = add(e)
	  }
	
	  class EnvelopePriorityComparator extends Comparator[dispatch.Envelope] {
	    def compare(o1: dispatch.Envelope, o2: dispatch.Envelope): Int = {
	      val priority1 = o1.message.asInstanceOf[Envelope[_]].attributes(PriorityAttribute).toString.toInt
	      val priority2 = o2.message.asInstanceOf[Envelope[_]].attributes(PriorityAttribute).toString.toInt
	      priority1 compareTo priority2
	    }
	  }
	
	}
	``` 