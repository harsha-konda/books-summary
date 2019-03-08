{{toc/}}

# Behavioural Patterns
## Chain of Resposibility
The chain of responsibility pattern creates a chain of receiver objects for a request. This pattern decouples sender and receiver of a request based on type of request. 

This achieved by having each reciever contain a reference to another reciever.If one object cannot handle the request then it passes the same to the next receiver and so on.

![uml](https://www.tutorialspoint.com/design_pattern/images/chain_pattern_uml_diagram.jpg)

### Java Implementation
* Abstract Class

```java

public enum Level { 	
    CONSOLE, ERROR, FILE; 
} 

public abstract class AbstractLogger {
    
    protected Level level;
	    protected AbstractLogger nextLogger;
	
	public void setNextLogger(AbstractLogger nextLogger){
	    this.nextLogger = nextLogger;
	}
		
	public void logMessage(Level level, String message){
	    if(this.level == level){
	        write(message);
	    } else if(nextLogger !=null){
	        nextLogger.logMessage(level, message);
	    } else {
	    	throw Exception;
	    }   
   }
}
```

* Console Class

```java
public class ConsoleLogger extends AbstractLogger {
   
   public ConsoleLogger(){
      this.level = Level.CONSOLE;
   }

   @Override
   protected void write(String message) {
      System.out.println("Standard Console::Logger: " + message);
   }
}
```

* Error Class

```java
public class ErrorLogger extends AbstractLogger {

   public ErrorLogger(){
      this.level = Level.ERROR;
   }

   @Override
   protected void write(String message) {
      System.out.println("Error Console::Logger: " + message);
   }
}
```

* File Logger Class

```java
public class FileLogger extends AbstractLogger {

   public FileLogger(){
      this.level = Level.FILE;
   }

   @Override
   protected void write(String message) {
      System.out.println("File::Logger: " + message);
   }
}
```

* Usage

```java
public class ChainPatternDemo {
	
    private static AbstractLogger getChainOfLoggers(){

	AbstractLogger errorLogger = new ErrorLogger();
	AbstractLogger fileLogger = new FileLogger();
	AbstractLogger consoleLogger = new ConsoleLogger();

	errorLogger.setNextLogger(fileLogger);
	fileLogger.setNextLogger(consoleLogger);

	return errorLogger;		
	}
	
   public static void main(String[] args) {
      AbstractLogger loggerChain = getChainOfLoggers();

      loggerChain.logMessage(Logger.INFO, 
         "This is an information.");

      loggerChain.logMessage(AbstractLogger.ERROR, 
         "This is an error information.");
   }

}	
```

### Scala Implementation
* Create case object for the enums

```scala
package LoggerLevel {
  sealed trait Level
  case object ERROR extends Level
  case object CONSOLE extends Level
  case object FILE extends Level
}

case class Request(loggerLevel: Level, message: String)

type Logger = PartialFunction[Request, Unit]

```

* Create the loggers

```scala
val fileLogger: Logger = {
    case Request(FILE,message) => println("File::Logger: " + message)
}
val errorLogger: Logger = {
    case Request(ERROR,message) => println("Error::Logger: " + message)
}

val consoleLogger: Logger = {
    case Request(CONSOLE,message) => println("Console::Logger: " + message)
}

```

*  Create handler that delegates it to the right logger


```scala

//default behaviour
val defaultLogger: Logger = PartialFunction(_ => ())

val logger = 
  fileLogger.orElse(errorLogger).orElse(consoleLogger).orElse(defaultLogger)

```

### FAQ
-  Okay, this seems like a contrived example, give me a real example where this would be necessary ?<br/>
   While designing akka actors, this is a very useful pattern as you would chain the functionalities and try to apply the right function for a given message.
   
- Which scenrio fits the ideal use case of chain of responsibility?<br/> Use the Chain of Responsibility pattern when you can conceptualize your program as a chain made up of links, where each link can either handle a request or pass it the next element in the chain.

- | Chain of Responsibility   | Decorator   | 
|---|---|
|Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it.|  Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality. |  

- What's a partial function?<br/> 

```
val fraction = new PartialFunction[Int, Int] {
  def apply(d: Int) = 42 / d
  def isDefinedAt(d: Int) = d != 0
}

fraction.isDefinedAt(42) // => true
fraction.isDefinedAt(0) // => false

scala> fraction(42)
res4: Int = 1
scala> fraction(0)
java.lang.ArithmeticException: / by zero

```   

also be written as 

```
val fraction: PartialFunction[Int, Int] =
  { case d: Int if d != 0 ⇒ 42 / d }
  
```

   