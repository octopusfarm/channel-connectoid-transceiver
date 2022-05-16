# channel-connectoid-transceiver
An abstract communication mechanism which makes calling a remote service very similar to calling a method. Relieves business logic of having to care about details like "HTTPS" and "JSON." You send an array (ordered map) and another comes back. That's it!

The design references `ResultNotification`, which you can find [here](https://github.com/octopusfarm/result-notification).

I've been using this pattern for years, and it has served me well. It's not the same as Guzzle or cURL, although it could use either as a back-end while communicating over HTTPS.
(That is the usual use case: JSON or XML over HTTPS.) However, Channel is not specific to HTTPS or any other pipe. It can talk to anything over TCP, UDP, a UNIX socket, or via method call.

If you have a monolith and want to peel off one of its services to run as a separate microservice, you could do this:

- Set up Channel to call whatever method your service is behind. Make everything that calls that method use Channel to talk to it instead.
- Begin porting your service over to its new microservice instance. Set up Channel so that it can talk to the new microservice under a different service name.
- Using configuration (a feature flag or whatever), switch your client code from talking to the local service to the remote microservice. If it breaks, you can switch it back the same way.
- When you're ready, retire the old local version of your service, as well as the Channel config that talks to it.

This is something I put together in a few hours, so the diagram doesn't have every detail (like how the config works, or how retry and failover would work.)
