# LUA_rateLimiting
An example of sliding window rate limiting using LUA scripting

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

Which can then be used with the EVALSHA command from the client along with necessary arguments which are:
# number of KEYS 
# KEYS[1] == general resource key
# KEYS[2] == consumer for resource key
# ARGV[1] == time window to govern measured in seconds
# ARGV[2] == size of allowed operations count for general resource key 
# ARGV[3] == size of allowed operations count for consumer for resource key


```
EVALSHA 6efc874244ac8797c2b893fdafcd95c25a480003 2 z:rl:tw:1sec:resource:12{12} z:rl:tw:1sec:resource:12:{12}consumer:1 10 5 3
```

Which results in output like this:

```
"Adding 1 point to your count and 1 point to the resource count"
```
