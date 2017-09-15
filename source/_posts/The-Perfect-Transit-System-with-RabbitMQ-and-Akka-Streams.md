---
title: 'The Perfect Transit System with RabbitMQ and Akka Streams'
date: 2017-09-09 14:27:46
banner:
  url: /images/akka-rabbit.png
  width: 640
  height: 360
tags:
---
Sitting in the train with no A/C stuck in the transit tunnel for more than 30 minutes. I yelled "I can build a better transit system than this!"

Of course, I didn't yell this out aloud in public transit, I kept it in my head. I am pretty sure that yelling out loud in public transit is guaranteed to freak everybody out. Anywho, I continued to think about actually trying to build out a transit system with the technologies I have worked with. The first thing that came to mind is that you could probably build a transit system with [Akka](http://akka.io/) and [RabbitMQ](https://www.rabbitmq.com/).

> Akka is a toolkit for building highly concurrent, distributed, and resilient message-driven applications for Java and Scala

If you think about it, a transit system is heavily reliant on communication between multiple different trains, stations, track sensors etc. All these system components are distributed in different areas of a city and messages are constantly being sent with the chance that messages may not reach their intended destination.

> RabbitMQ...supports multiple messaging protocols...can be deployed in distributed and federated configurations to meet high-scale, high-availability requirements.

As you can tell, RabbitMQ would be complimentary to Akka and supports some of the same use cases Akka was built for. Also, using RabbitMQ allows us to connect heterogeneous systems with vastly different messaging protocols.


Alright so how do we even start building something like this and where do we start? Well...we could start with a vehicle!

## Vehicle Dispatching
[Getting started with Akka/Scala](http://doc.akka.io/docs/akka/current/scala/guide/introduction.html?_ga=2.127584205.1701460921.1504998808-783609866.1497769625#how-to-get-started) is pretty simple so I won't go over how that is done. An important thing to keep in mind is that Akka is based on the [Actor Model](https://en.wikipedia.org/wiki/Actor_model). It is a different way of dealing with a concurrency within a system. Actors receive messages from other actors asynchronously. They may process a message from another actor and reply back or they may choose not to.  It feels like OOP...but supercharged and asynchronous. Akka provides a lot of capabilities for you to build a whole system of actors. The first actor we are going to create is a Dispatcher actor. It is going to be in charge of receiving dispatch requests from an external source and then create vehicles that will be managed in the transit system.

```scala
class Dispatcher extends Actor
  with ActorLogging {

  override def receive = {
      case dispatchRequest: DispatchVehicle =>
        log.info(s"Vehicle Dispatched: $dispatchRequest")
      case request:InfoRequest => {
        log.info(s"Requesting vehicle info for: ${request.vehicleId}")
      }
      case _ => log.info(s"unhandled request")
    }
}

object Dispatcher {
  final case class DispatchVehicle(
  name: String, 
  routeName: String,
  direction: String)
  
  final case class InfoRequest(vehicleId: String)
  
  val props: Props = Props[Dispatcher]
}
```

Alright so what in tarnation is going on here... 

Line 1: All actors extend from the Actor trait. This trait requires that you implement the  `def receive` method which is a method that receives async messages (A message could be anything that is typed in Scala, so think of Strings, Int or any other class or object type) from some external source.

Line 4 - 13: This is were the actor does the heavy lifting. In here, you define the behavior of an actor based on a number of different cases. This uses Scala's Pattern Matching to determine what kind of message it receives. Depending on the type of message you can specify your actor's behavior

Line 15 - 22: This is an companion object for the `Dispatcher` class. It holds the message classes that the `Dispatcher` actor is aware of. Also pay attention to Line 20. This is a factory method used to create new actors. It is in the companion object of `Dispatcher` The Akka Docs [here](http://doc.akka.io/docs/akka/current/scala/actors.html#recommended-practices) explain why it is a good idea do it this way. When an actor is created, it get what is called an `ActorRef`. This is sort of like an identifier used to locate an Actor within an ActorSystem. _THIS IS VERY IMPORTANT_ as it is vital when trying to identify actors to send messages too.

The Akka documentation recommends that care is needed in managing the lifecycle of all actors in an actor system. Documentation suggests that an actor system needs to resemble a tree structure of some sort like so ![Actor Model](http://arild.github.io/akka-workshop/figures/actor-model.png "Image Source: http://arild.github.io/akka-workshop/figures/actor-model.png")
What this basically means is that you need to create actors in the context of other actors (parent/child relationship) and that parent actors dictate the lifecycle of child actors. We will see this later on. 

## Firin' Up the Actor System
This is great and all but we want to see how this actor actually works! To do that we have to have a basic Akka project set up. You can download a quick start project and modify it the way you like [here](http://dev.lightbend.com/start/?group=akka&project=akka-quickstart-scala) and instructions to get the project up and running can be found [here](http://developer.lightbend.com/guides/akka-quickstart-scala/?_ga=2.52359849.1894541004.1505449743-783609866.1497769625)

If you followed the akka-quickstart-scala project you may begin to see how the Akka kinda works. I went ahead and modified my `AkkaQuickstart.scala` and renamed it to `Main.scala`. I also modified the innards of the `Main.scala` to do the following

1. Create our actor system. This house all the actors that we will be using to help manage vehicles within our tranportation system
2. Create our first `Dispatcher` Actor so that we can send it messages
3. Send 3 different type messages of `DispatchVehicle`, `InfoRequest` and a `String` message type to observe how our `Dispatcher` reacts.*
4. We finally terminate the system with `system.terminate()` This shuts down all actors and the system itself. Ain't nobody got time for memory leaks.

```scala
object Main extends App {
  val system: ActorSystem = ActorSystem("perfecttransit")
  try{
    val dispatcherRef = system.actorOf(Dispatcher.props, "dispatcher")
    dispatcherRef ! DispatchVehicle("123B","J Line", "Inbound")
    dispatcherRef ! InfoRequest("Vehicle#1")
    dispatcherRef ! "Bogus"
  } finally {
    system.terminate()
  }
}
```

*Pay close attention to Line 5. The statement `dispatcherRef ! DispatchVehicle("123B","J Line", "Inbound")` Is using the `tell` operator to send a `DispatchVehicle` message to the Dispatcher actor. The `tell` operator takes an `ActorRef` to the left and a message to the right. Remember the sending of messages is asynchronous so there is no return information. we are just sending a message to `Dispatcher` and expecting it will do something appropriate with it.

Go ahead and run `Main` using `sbt run` in your main project folder within your terminal. What you should see after you run the project are log messages showing how the `Dispatcher` actor behaves.

We know our Vehicle Transit System the capability of receiving messages but we want to be able to receive messages externally from other sources. This is were we are going to take advantage of RabbitMQ!