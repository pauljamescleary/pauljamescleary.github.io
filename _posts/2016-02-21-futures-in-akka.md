---
layout: post
title: Futures in Akka
description: "Fun times using Futures from Actors"
modified: 2016-02-20
tags: [future, akka, actor, failure, supervision]
image:
  feature: abstract-4.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
---

_Full Gist Here - [https://gist.github.com/pauljamescleary/fe4a2888415ea4b8d184](https://gist.github.com/pauljamescleary/fe4a2888415ea4b8d184)_

Handling the results of Futures inside of Akka Actors may not be as trivial as you would think.

The following is a snippet that would lookup a Jawn from a database and pipe it 
to the sender

~~~scala
case class Jawn(id: String)

trait Repo {
  def getJawn(id: String): Future[Option[Jawn]]
}

object FutureActor {
  case class GetJawn(id: String)
}

class FutureActor(repo: Repo) extends Actor {
  import FutureActor._
  
  def receive = {
    case GetJawn(id) =>
      println("...getting jawn...")
      repo.getJawn(id) pipeTo sender
  }
}

class ParentActor extends Actor {

  implicit val timeout = Timeout(1.second)

  val okRepo = new Repo {
    def getJawn(id: String): Future[Option[Jawn]] =
      Future {
        Some(Jawn(id))
      }
  }

  val futureActor = context.actorOf(Props(classOf[FutureActor], okRepo))

  def receive = {
    case str: String =>
      val result1 = Await.result(futureActor ? GetJawn(str), 1.second)
      println(s"$result1;")
  }
}
~~~

All of this looks reasonable.  We can bootstrap a simple main class and see what happens when it runs

~~~scala
object FutureActorRunner extends App {

  val system = ActorSystem("testing")

  val parent = system.actorOf(Props(classOf[ParentActor]))

  parent ! "anyThing"
}

~~~

And when we run, we will see the output:

~~~
...getting jawn...
Some(Jawn(anyThing))
~~~

The question is, what happens when the database fails?  Under most circumstances we would like to invoke supervision on our Actor.  Shit, the database is down so our 
actor is likely not going to do what you ask of him.  We really don't want to continue sending messages to our Actor while the database is down.

Let's take this one step further and cause a failure.  What will happen to our Actor?

Let's make some subtle changes to our parent actor.  

- First, we will implement a *BadRepo* that will always fail.  
- Second, we will send two messages to our Parent to see what happens.  Here are some small code snippets that highligh the changes:

~~~scala
class ParentActor extends Actor {
  ...
  val badRepo = new Repo {
    def getJawn(id: String): Future[Option[Jawn]] =
      Future.failed(new RuntimeException("bad boy!"))
  }

  val futureActor = context.actorOf(Props(classOf[FutureActor], badRepo))
  ...

  parent ! "anyThing"
  parent ! "chumpie"
~~~

Look at the output now:

~~~
...getting jawn...
...getting jawn...
~~~

Well that doesn't seem good.  We continue to attempt to get the jawn even though our ficitious database is down.  So, it appears as though that our failure is not invoking supervision.

## A Sidebar on Supervision
What does this supervision business mean anyway?

In Akka (and any Actor system), one of the greatest features is being able to detect errors in the system, and respond to failure accordingly.

There are several things that can be done when an issue is detected:

- *Resume* - this basically means that we just let the actor go on its merry way.  Perhaps we log the issue, or keep a counter, whatevs.  But the exception should not invoke any response
- *Stop* - stop our Actor from doing anything.  Anyone who happens to be watching the Actor will see a *Terminated* message
- *Restart* - restart the Actor essentially clears out its current state, but keeps the mailbox for the Actor intact.
- *Escalate* - shit, I have no idea what to do, let my parent decide. 

All exceptions are handled by the _Parent Actor_ (i.e. the actor that actually created the child) in a *SupervisorStrategy*.  The Default strategy is a *OneForOneStrategy*; meaning that supervision will treat each actor separately.  The default strategy uses the following *Decider* to determine what to do when an exception arrives:

~~~scala
    case _: ActorInitializationException ⇒ Stop
    case _: ActorKilledException         ⇒ Stop
    case _: DeathPactException           ⇒ Stop
    case _: Exception                    ⇒ Restart
~~~

If you do nothing, then by default the Actor will be restarted when an exception comes in.

## Back to our example
Well, with default supervision in place, then maybe something _is_ happening afterall!?  Maybe we are restarting and we don't see it anywhere.

Let's make another change to child actor.  Let's add a lifecycle hook to catch the `Stop` of the actor.  *postStop* is called when the actor restarts.  Given that the default strategy is to restart him, this should get called...

~~~scala
class FutureActor ...

  override def postStop(): Unit = {
    println("...Future Actor Stopping...")
    super.postStop()
  }
}
~~~

Now that we have that change in, we should see it being invoked on failure, let's look at our output again to see if we can spot our trace statement...

~~~
...getting jawn...
...getting jawn...
~~~

*WHAT!*  You mean we didn't restart.  What is going on!

Rather than keeping you in suspense any longer, I will give you the skinny.  
When you kick off a Future from an Actor, we say that the Future "executes outside of the actor".  That means that it really has no place to send the results to because it will complete "in the future".

Actors _only_ work by processing messages on their mailbox.  They just don't sit around waiting for Futures to complete.  We _could_ block using `Await.result`, but that is really shady as we could fill the Mailbox for our Actor if a lot of messages are coming in while we are waiting for results.

The _only_ way to process the results of a Future from an Actor is to put a message back on the Actor's mailbox.

This is where sadness ensues, but it is a necessary evil AFAIK.

## Updating our app to handle failures
Let's again modify our little example.  This time, we are going to make the following changes:

- Intercept the result of the Future.  If it succeeds, then send the results to the sender.  If it fails, then send a failure to ourselves.
- Handle the *akka.actor.Status.Failure* message, and throw the error that arrives.

Here is the new and improved *FutureActor*

~~~scala
class FutureActor(repo: Repo) extends Actor {

  import FutureActor._

  def receive = {
    case GetJawn(id) =>
      println("...getting jawn...")
      repo.getJawn(id).onComplete {
        case Success(jawn) => sender ! jawn
        case Failure(e) => self ! akka.actor.Status.Failure(e)
      }

    case akka.actor.Status.Failure(e) =>
      throw e;
  }

  override def postStop(): Unit = {
    println("...Future Actor Stopping...")
    super.postStop()
  }
}
~~~

We will run our test again, and checkout the output:

~~~
...getting jawn...
...Future Actor Stopping...
...getting jawn...
...Future Actor Stopping...
~~~

Success!  We have forced our actor to restart (we can see the "Stopping" message in the output)

## Conclusions
Yea, I no like this.  There are ways to get around this, because it feels like alot of boilerplate coming our way.

Options include:

- Creating your own Actor base class that overrides *unhandled*.  That would at least keep you from having to check *akka.actor.Status.Failure* in all your actors
- Some kind of concoction that would auto send failures to the actor.

I am working on solutions.  Certainly, if you all have experience with this problem and have developed an elegant solution (or can tell me where I went wrong), but all means, please provide feedback!








