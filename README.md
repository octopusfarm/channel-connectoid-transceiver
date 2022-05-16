# channel-connectoid-transceiver
An abstract communication mechanism which makes calling a remote service very similar to calling a method. Relieves business logic of having to care about details like "HTTPS" and "JSON." You send an array (ordered map) and another comes back. That's it!

The design references `ResultNotification`, which you can find [here](https://github.com/octopusfarm/result-notification).

I've been using this pattern for years, and it has served me well. It's not the same as Guzzle or cURL, although it could use either as a back-end while communicating over HTTPS.
(That is the usual use case: JSON or XML over HTTPS.) However, Channel is not specific to HTTPS or any other pipe. It can talk to anything over TCP, UDP, a UNIX socket, a queue (like SQS FIFO or Kafka), or (as described below) via method call.

If you have a monolith and want to peel off one of its services to run as a separate microservice, you could do this:

- Set up Channel to call whatever method(s) serve as the interface to your service. Make everything that calls those methods use Channel to talk to it instead. At this point, nothing is going over the wire.
- Begin porting your service over to its new microservice instance. In the Channel configuration (which is just an array returned by a PHP file), use a feature flag to switch behavior between "call the methods directly" and "call the methods on the new microservice over HTTPS or whatever."
- Using the feature flag, switch your client code from talking to the local service to the remote microservice. If it breaks, you can switch it back simply by changing the feature flag.
- When you're ready, retire the old local version of your service, as well as the Channel config that talks to it. Make sure the remaining monolith can't authenticate to the new microservice's database, as it has no business directly modifying data that now belongs exclusively to that microservice.

This is something I put together in a few hours, so the diagram doesn't have every detail (like how the config works, or how retry and failover would work.)
