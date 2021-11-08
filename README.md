# LUA_rateLimiting
2021-11-07 

An example of 'sliding window rate limiting' using LUA scripting.

The purpose of the solution is to keep track of how many invocations are made both against a shared resource in total, as well as for every individual consumer of that resource.  

By keeping track we can also enforce a limiting behavior - call this script before you invoke the actual underlying resource and if you get a positive response go ahead and invoke the actual resource.  If you get a negative response indictaing that either the reource is over the limit, or the consumer is over its limit, then you must wait a bit before trying again. 

This script uses SortedSets that store timestamps of each invocation made.
The SortedSet is a collection of entries with 2 values for each entry:
1) the score of the entry
2) the value of the entry

In our example, the SortedSets get populated with the timestamp up to the most recent second as the score and the value of the microseconds that have passed since the most recent second as the value. (this value is used to ensure that there are very few collisions within a single key - as SortedSets will not allow multiple entries with the same value)

Here is the result of calling ZRANGE on a key that holds entries for a shared resource with an id of 12:

```
127.0.0.1:6379> ZRANGE z:rl:tw:1sec:resource:12{12} 0 -1 withscores
1) "104768"
2) "1636330091"
3) "111348"
4) "1636330093"
5) "129134"
6) "1636330097"
```

The script acts in this way:

When invoked, it deletes all records from the shared resource key that are older than the desired time window.  (In our example below, it delete all records older than 10 seconds before the moment of invocation).
  
The script next checks the number of remaining items stored in the SortedSet for the shared resource key and compares that number to the allowed number of invocations for that resource for the desired time-window.  
If the number is too large, the shared resource is over the limit and a negative message is returned to the caller.   
If the number for the shared resource is not too large, the script then deletes all records from the individual consumer key that are older than the desired time window.
The script then checks the number of remaining items stored in the SortedSet for the individual consumer (of the resource) key and compares that number to the allowed number of invocations for that consumer for the desired time-window.  
If the number is too large, the consumer is over its limit and a negative message is returned to the caller.   

If neither the resource nor the consumer are over the limit for the time window, both keys get an entry written to them that captures the moment of invocation so that future queries will count them against the next time window.  This is a positive outcome for the caller who should then proceed to invoke the actual desired resource as no limits have been reached by that caller.


The script follows:

```
SCRIPT LOAD "local rlimit = 0+ARGV[2] local climit = 0+ARGV[3] local t = (redis.call('TIME'))[1] local t2 = (redis.call('TIME'))[2] redis.call('ZREMRANGEBYSCORE',KEYS[1],'0',(t-ARGV[1])) local rcount = (redis.call('ZCARD',KEYS[1])) if (rcount == rlimit) then return 'RESOURCE OVERLIMIT: '..redis.call('ZCARD',KEYS[1]) elseif (((redis.call('ZREMRANGEBYSCORE',KEYS[2],'0',(t-ARGV[1]))) < 1000000) and (redis.call('ZCARD',KEYS[2]) < climit)) then return 'Adding '..(redis.call('ZADD',KEYS[2],t,t2))..' point to your count and '..(redis.call('ZADD',KEYS[1],t,t2))..' point to the resource count' else return KEYS[2]..' CONSUMER OVERLIMIT: '..redis.call('ZCARD',KEYS[2]) end"
```

You can load it into redis and receive a SHA unique id that can be later used to invoke it without passing the entire script every time:

```
SCRIPT LOAD "local rlimit = 0+ARGV[2] local climit = 0+ARGV[3] local t = (redis.call('TIME'))[1] local t2 = (redis.call('TIME'))[2] redis.call('ZREMRANGEBYSCORE',KEYS[1],'0',(t-ARGV[1])) local rcount = (redis.call('ZCARD',KEYS[1])) if (rcount == rlimit) then return 'RESOURCE OVERLIMIT: '..redis.call('ZCARD',KEYS[1]) elseif (((redis.call('ZREMRANGEBYSCORE',KEYS[2],'0',(t-ARGV[1]))) < 1000000) and (redis.call('ZCARD',KEYS[2]) < climit)) then return 'Adding '..(redis.call('ZADD',KEYS[2],t,t2))..' point to your count and '..(redis.call('ZADD',KEYS[1],t,t2))..' point to the resource count' else return KEYS[2]..' CONSUMER OVERLIMIT: '..redis.call('ZCARD',KEYS[2]) end"
```

This will return a hash like this:

```
"6efc874244ac8797c2b893fdafcd95c25a480003"
```

Which can then be used with the EVALSHA command along with necessary arguments which are:
### number of redis KEYS that will be passed in ( a neccessary part of using Lua with redis )
### KEYS[1] == shared resource key
### KEYS[2] == consumer for resource key
### ARGV[1] == time window to govern measured in seconds
### ARGV[2] == size of allowed operations count for shared resource key 
### ARGV[3] == size of allowed operations count for consumer for resource key

* in the following example the resource '12' has a limit of 5 invocations every 10 seconds, while the individual consumer '1' has a limit of 3 operations every 10 seconds. 

```
EVALSHA 6efc874244ac8797c2b893fdafcd95c25a480003 2 z:rl:tw:1sec:resource:12{12} z:rl:tw:1sec:resource:12:{12}consumer:1 10 5 3
```

Which results in output like this:

```
"Adding 1 point to your count and 1 point to the resource count"
```

If you run the command many times you will hit the limit for that consumer and get a response:

```
"z:rl:tw:1sec:resource:12:consumer:1 CONSUMER OVERLIMIT: 3"
```

If you run a few instances of the command where each one uses a different consumer key, they will, as a group hit the limit for the resource they share and get a response like this:

```
"RESOURCE OVERLIMIT: 5"
```

An easy way to demonstrate these behaviors is to use the redis-cli with the -i (for interval) argument.
Start the first redis-cli with a 1 second interval:

```
redis-cli -i 1
```

Then at the prompt issue the call to the script using the first consumer id:  [note that I also use a number before the call to ask redis-cli to repeat the command 100 times]

```
100 EVALSHA 6efc874244ac8797c2b893fdafcd95c25a480003 2 z:rl:tw:1sec:resource:12{12} z:rl:tw:1sec:resource:12:{12}consumer:1 10 5 3
```

In a separate shell, start the second redis-cli instance with a 2 second interval:

```
redis-cli -i 1
```

Then at the prompt issue a call to the script using a different consumer id: (this is done by changing the name of the second key passed in) 

```
100 EVALSHA 6efc874244ac8797c2b893fdafcd95c25a480003 2 z:rl:tw:1sec:resource:12{12} z:rl:tw:1sec:resource:12:{12}consumer:2 10 5 3
```


With a resource limit of 5 operations every 10 seconds, it is easy to see how both the resource limitation gets triggered, as well as the consumer limitations.

Output from the first redis-cli session looks like this once the second session gets going:

```
"Adding 1 point to your count and 1 point to the resource count"
"Adding 1 point to your count and 1 point to the resource count"
"Adding 1 point to your count and 1 point to the resource count"
"RESOURCE OVERLIMIT: 5"
"z:rl:tw:1sec:resource:12:{12}consumer:1 CONSUMER OVERLIMIT: 3"
"RESOURCE OVERLIMIT: 5"
"z:rl:tw:1sec:resource:12:{12}consumer:1 CONSUMER OVERLIMIT: 3"
"RESOURCE OVERLIMIT: 5"
"RESOURCE OVERLIMIT: 5"
"RESOURCE OVERLIMIT: 5"
"Adding 1 point to your count and 1 point to the resource count"
"Adding 1 point to your count and 1 point to the resource count"
"Adding 1 point to your count and 1 point to the resource count"
"RESOURCE OVERLIMIT: 5"
"z:rl:tw:1sec:resource:12:{12}consumer:1 CONSUMER OVERLIMIT: 3"
"RESOURCE OVERLIMIT: 5"
"z:rl:tw:1sec:resource:12:{12}consumer:1 CONSUMER OVERLIMIT: 3"
"RESOURCE OVERLIMIT: 5"
"RESOURCE OVERLIMIT: 5"
"RESOURCE OVERLIMIT: 5"
```

Meanhwile, the slower second session with the 2 second interval between its invocations shows this kind of output:

```
"Adding 1 point to your count and 1 point to the resource count"
"Adding 1 point to your count and 1 point to the resource count"
"RESOURCE OVERLIMIT: 5"
"RESOURCE OVERLIMIT: 5"
"RESOURCE OVERLIMIT: 5"
"Adding 1 point to your count and 1 point to the resource count"
"Adding 1 point to your count and 1 point to the resource count"
"RESOURCE OVERLIMIT: 5"
"RESOURCE OVERLIMIT: 5"
"RESOURCE OVERLIMIT: 5"
```

Looking at the resulting cardinality of the involved SortedSets reveals this state at the end of the run:

```
127.0.0.1:6379> ZCARD z:rl:tw:1sec:resource:12{12}
(integer) 5

127.0.0.1:6379> ZCARD z:rl:tw:1sec:resource:12:{12}consumer:1
(integer) 3

127.0.0.1:6379> ZCARD z:rl:tw:1sec:resource:12:{12}consumer:2
(integer) 3
```


## A version of the script that only returns True when it is OK to invoke the shared resource and False when it is not OK looks like this:

```
"local rlimit = 0+ARGV[2] local climit = 0+ARGV[3] local t = (redis.call('TIME'))[1] local t2 = (redis.call('TIME'))[2] redis.call('ZREMRANGEBYSCORE',KEYS[1],'0',(t-ARGV[1])) local rcount = (redis.call('ZCARD',KEYS[1])) if (rcount == rlimit) then return 'False' elseif (((redis.call('ZREMRANGEBYSCORE',KEYS[2],'0',(t-ARGV[1]))) < 1000000) and (redis.call('ZCARD',KEYS[2]) < climit)) then redis.call('ZADD',KEYS[1],t,t2) redis.call('ZADD',KEYS[2],t,t2) return 'True' else return 'False' end"
```

Here is an example run with a 1/second redis-cli:

```
127.0.0.1:6379> 25 EVALSHA d64787ff546895181f231ed3102003a9697a7704 2 z:rl:tw:1sec:resource:12{12} z:rl:tw:1sec:resource:12{12}:consumer:1 10 5 3
"True"
"True"
"False"
"True"
"False"
"False"
"False"
"False"
"False"
"False"
"True"
"True"
"False"
"True"
"False"
"False"
"False"
"False"
"False"
"False"
"True"
"True"
"False"
"True"
"False"
(25.16s)
```

And the same time-frame with a 2/second redis-cli:

```
127.0.0.1:6379> 10 EVALSHA d64787ff546895181f231ed3102003a9697a7704 2 z:rl:tw:1sec:resource:12{12} z:rl:tw:1sec:resource:12{12}:consumer:2 10 5 3
"True"
"True"
"False"
"False"
"False"
"True"
"True"
"False"
"False"
"False"
(20.06s)
```

And the resulting ZCARD data for the keys involved:

```
127.0.0.1:6379> zcard z:rl:tw:1sec:resource:12{12}:consumer:2
(integer) 2
(1.00s)
127.0.0.1:6379> zcard z:rl:tw:1sec:resource:12{12}:consumer:1
(integer) 3
(1.00s)
127.0.0.1:6379> zcard z:rl:tw:1sec:resource:12{12}
(integer) 5
(1.00s)
```

