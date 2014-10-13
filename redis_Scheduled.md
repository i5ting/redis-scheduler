# Building a Scheduled Queue With Redis

http://blog.codezuki.com/blog/2013/07/07/redis-queue

Something that comes up every now and again is, “How would I build a distributed, scheduled queue?”.

Lets say I have an online order system that can accept two types of orders, those that must be processed immediately and others which can be scheduled to be processed at some arbitrary time in the future.

One approach might be to store all of the orders in an enterprise RDMS and poll it frequently to find orders whose schedule date is now met. If there was a huge amount of data this could put an unneccessary strain on the database.

## Redis to the rescue!

Redis can be used to act as a queue of orders. There will be two queues, the “Process Now” queue and the “Process Later” queue. On these queues I will store just the ORDER_ID, the full order details are still stored in our RDBMS. Everything in the “Process Now” queue is ready to be processed, so we can just pop the next one off that queue (using BLOP) and perform billing and order fulfilment. This queue is just a Redist List that we are treating like a queue. The orders in the “Process Later” queue must be processed when the order’s schedule date has been met. This queue is simply mapping a key to an empty value with an expiration date, it’s not even really a list.

Redis可以扮演订单队列的角色。有2个队列，立即处理和延时处理。

How do we know the schedule has been met without searching the queue? An upcoming feature of Redis is Keyspace Notifications. Amongst other things, this means we can subscribe to the expired key event. Lets be clear, Redis must be searching for keys to expire and the documentation clearly states there is an overhead associated with enabling Keyspace Notifications, so this feature is not free in terms of CPU time but it releases some load from the RDBMS.

## Get Redis

Unfortunately Keyspace Notifications are currently only available in development versions of Redis. The examples I toyed with here worked with a development build of Redis 2.8, and here’s how I got it working.

### Clone Redis from Github

Go to the [Redis GitHub page](https://github.com/antirez/redis) and clone the repository, then checkout branch 2.8 and follow the instructions in the [README](https://github.com/antirez/redis/blob/unstable/README) file to build Redis locally. It’s very easy to do and doesn’t take long.

### Enable Keyspace Notifications

Because keyspace event notification carries an overhead, you have to enable it as an option when Redis starts. I did so like this.

	./redis-server --notify-keyspace-events Ex # E = Keyevent events, x = Expire events


You can also do this permanently by editing the redis.conf file, which also contains further comments about other events you may be interested in. It is worth reading those comments if you want to play with this feature.

The Redis website has a [topic on notifications](http://redis.io/topics/notifications).


## Schedule and observe expiring keys

For simplicity I’m using the native Redis Client to demonstrate these examples. There is a complete node.js example at the end of this post.

Open two Redis Clients in separate terminals.


	./redis-cli

In the first client schedule an order id in the “Process Later” queue. Use an expiry time of 10 seconds in the future to give us enough time to switch to the other terminal and watch the magic!


	set order_1234 '' PX 10000 # set the key 'order_1234' with it's expiry time in 10000ms
	
Here we are not interested in the value, just the key order_1234 and that it expires in 10000ms. The expiry time would be the number of milliseconds in the future until the order fulfilment date.


More on the Redis command [SET](http://redis.io/commands/set).

### Subscribe to the key expiring events

In the second client subscribe to be notified when keys expire like this.

	psubscribe '__keyevent@0__:expired'
	
More on the Redis command [PSUBSCRIBE](http://redis.io/commands/psubscribe).

Now we are subscribers to the queue of “Process Later” orders. We will be notified when the schedule date is reached in the form of an event telling us that the ORDER_ID has expired. This is nice as the key is automatically being removed from the “Process Later” queue too. Now all we have to do is place the expiring ORDER_ID onto the “Process Now” queue and the order will be fulfilled.

This is what the notification looks like when the key expires.

	1) "pmessage"
	2) "__keyevent@0__:expired"
	3) "__keyevent@0__:expired"
	4) "order_1234"


## Making it distributed

You can do a lot with Redis running on a single server, but if we wanted to make this fault tolerant to network partitions and scale horizontally then we would probably look at sharding and replication. Some clients support automatic sharding using [consistent hashing](http://michaelnielsen.org/blog/consistent-hashing/) which would work well with our `ORDER_ID` in this example.

The Redis website has a topic on [partitioning](http://redis.io/topics/partitioning).

Here is a good post about [replication with Redis](http://aphyr.com/posts/283-call-me-maybe-redis).


## An example in node.js

Here is a crude but working example I created as a GitHub Gist.


	var redis = require("redis"),
 
	// create a client to subscribe to the notifications
	subscriberClient = redis.createClient(),
 
	// create a client which adds to the scheduled queue
	schedQueueClient = redis.createClient();
 
	// when a published message is received...
	subscriberClient.on("pmessage", function (pattern, channel, expiredKey) {
	    console.log("key ["+  expiredKey +"] has expired");
 
	    //TODO: push expiredKey onto some other list to proceed to order fulfillment
 
	    subscriberClient.quit();
	});
 
	// subscribe to key expire events on database 0
	subscriberClient.psubscribe("__keyevent@0__:expired");
 
	// schedule ORDER_ID "order_1234" to expire in 10 seconds
	schedQueueClient.set("order_1234", "", "PX", 10000, redis.print);
	schedQueueClient.quit();


## Caveat emptor

Okay so here is the disclaimer. I have rather unfairly demonstrated a development build of Redis, so be aware that everything could change and although I encountered no problems, it is only reasonable to expect a pre-release quality experience.

