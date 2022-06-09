# channel-connectoid-transceiver

**What if sending requests was as straightforward as _receiving_ them?**

When your Controller receives a request, its contents have already been unserialized to a live data type: an object, an array, etc. You don't have to decode the JSON, XML, or whatever. Your controller works whether you call it over HTTPS, or from inside a unit test.

More to the point, it works the same either way _because_ it does't know whether it's being called over HTTPS or from inside a test! You can swap that out any time you like without breaking any of your code.

However, _after_ the request arrives at your Controller, at some point you have to talk to a service, and then you have stuff like this:

```
request = encodeMapToJSON(requestParams);
response = someFramework.sendRequestOverHTTPS(someURL, request);
```

What if you want to replace JSON with protobuf, and HTTPS with HTTP/2? What if the URL changes? Now you have to find all the places with code like this and modify them. This also means you'll need some feature flag boilerplate if you want to refactor a monolith's service out to a microservice.


**Channel-Connectoid-Transceiver**

This abstract communication mechanism makes calling a service very much like calling a method. Whether the service actually _is_ a method call, or a process running on the other end of a network connection using whatever encoding and transport, is invisible — _and that's the point._ You can swap out those details behind the scenes, even at runtime, without breaking whatever calls the service.

You send an arbitrarily-nested ordered map — an array in PHP, an object in JS, etc. — and another comes back.

Because this mechanism is built to assume that failure occurs from time to time, it returns responses that can tell you whether they're valid, what worked (if anything), and what errors occurred (if any.) This references `ResultNotification`, which you can find [here](https://github.com/octopusfarm/result-notification).

**Examples**
```
// Simplest way: One-line synchronous request.
$response = Channel::call('some.service-name', $requestParams);
$this->doSomethingWith($response);
```

```
// Slightly fancier: Request with its own response callback.
$request = new Request('some.service-name', $requestParams);
$request->then(function(Response $response) {
    $this->doSomethingWith($response);
});

// Blocking send-and-receive.
Channel::transceive($request);

// OR, non-blocking, and you can do other stuff while you wait for the response.
$handle = Channel::transmit($request);
// ...send another request, write to the DB, etc...
Channel::receive($handle);
```

```
// Very fancy: Send many (possibly thousands), fifteen at a time.
// Generate each Request just before it's needed, and discard it when it has been responded to. Hardly uses any RAM.
$generator = function(array $minimalParams) : Request {
    foreach($minimalParams as $param) {
        $request = new Request('some.service-name', [
            'interestingThing' => $param,
            'otherParams' => [
                // ...
            ]
        ]);
        $request->then(function(Response $response) {
            // When each Response is received, it lands here.
            // (This is often a good place to put a breakpoint.)
            $this->handleResponse($response);
        });
        yield $request;
    }
}

// Send however many requests the generator function will give us, but not more than 15 at a time.
// The runQueue() method blocks until the last response (or comm failure) has arrived.
Channel::runQueue($generator($minimalParams), concurrency: 15);
```

```
// When a response is returned, it should ALWAYS be processed with the assumption that it will fail sometimes!
function handleResponse(Response $response) {
    $result = $response->result();
    if($result->valid()) {
        $model = new SomeModel();
        $model->particularValue = $response->get('some.nested.value');
        $model->save();
    } else {
        // Comm and/or service errors will be in $result->errors()
    }
}
```

I've been using this pattern for years, and it has served me well. It's not the same as Guzzle or cURL, although it could use either as a back-end while communicating over HTTPS. (That is the usual use case: JSON or XML over HTTPS.) However, Channel is not specific to HTTPS or any other pipe. It can talk to anything over TCP, UDP, a UNIX socket, a queue (like SQS FIFO or Kafka), or (as described below) via method call.


**Refactoring Monoliths to Microservices**

If you have a monolith and want to peel off one of its services to run as a separate microservice (whether in the same language or another), you could do this:

- Set up Channel to call whatever method(s) serve as the interface to your service. Make everything that calls those methods use Channel to talk to it instead. At this point, nothing is going over the wire.
- Begin porting your service over to its new microservice instance. In the Channel configuration (which is just an array or object), use a feature flag to switch behavior between "call the methods directly" and "call the methods on the new microservice over HTTPS with JSON or whatever."
- Using the feature flag, switch Channel from calling the local service's methods, to calling the remote microservice. If this breaks, you can switch it back simply by changing the feature flag.
- When you're ready, retire the old local version of your service, as well as the Channel config that talks to it. Make sure the remaining monolith can't authenticate to the new microservice's database, as it has no business directly modifying data that now belongs exclusively to that microservice.

This is something I put together in a few hours, so the diagram doesn't have every detail (like how the config works, or how retry and failover would work.)
