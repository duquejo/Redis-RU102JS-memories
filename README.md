# Redis-RU101-memories

<p align="center"><b>RU101: Introduction to Data Structures</b></p>
<p align="center"><a href="https://university.redis.com" target="_blank"><img src="https://prod-amc-bucket.s3.amazonaws.com/customer_files/2_redis-university-reversedRGB.png" alt="Redis University" /></a></p>
<p align="center">Redis Developer Certification Walkthrough</p>

 ## Redis Node.JS clients
- Has a callback pattern implementation
- Has an async/await implementation
- Manage connections
- Implement the Redis Protocol and API.
- Provides A usable language-specific API:
- Redis must always connect over TCP protocol.
- They can also have a special code for pooling these connections for reuse.
- As Node.JS, Redis shares a single-threaded programming model. Polling is not usually a concern for NodeJS developers.

### Redis client ways to connect
- With port and host
- Default port and host (6379/127.0.01)
- URL
- Using password
- With SSL TLS (key/cert/ca)

### node-redis events
- **Ready**: Connection established, commands can be sent.
- **End**: Connection has been closed
- **Reconnecting**: Connection lost, and reconnection attempts are being made.

**When using Redis blocking commands with Node.JS, a second connection will be required.**
For the node_redis client, it’s necessary to promesify commands, because all methods have a callback pattern. So, after promising (through Bluebird or native 'promisify' Node JS util), the functions can be used as a single promise→then structure or async/await pattern.

## Redis Serialization Protocol (RESP)
Language that Redis clients uses to communicate with the Redis Server
- **node-redis** (Recommended, Generar purposes).
- **ioredis** (Recommended, more suitable for managing Lua Scripts).

## Type mappings
| Redis type | JS type |
| --- | --- |
| string | String |
| list | Array of String |
| set | Array of String |
| Hash | Object (keys va String values) |
| float (INCRBYFLOAT) | String |
| integer (INCR) | number |

## DAO design pattern overview
Data Access Object (DAO), Separates the data access interface from the logic for interacting with a given data store.
Allows for multiple storage implementations.
- Domain Objects: Pure data representations
- DAO Interfaces: Data-store-agnostic API
- DAO implementations: Interact with a particular data Store.

## Key management best practices:
- Manage all keys from a single point in the code.
- Minimize maintenance when changing key names.
- Use an application-wide key name prefix as a namespace.
    - `ru102js:sites:ids`
- _DRY (Don’t repeat yourself)_.

## Advantages of data-structures

### Advantages of a sorted set
- Measurements are always sorted
- Efficiently fetch a small range `O((log n) + m)`
- Efficient inserts: `O(log n)`
- Multiple measurements per key save memory

### Advantages of Lua Script
- Cache all the scripts that are executed.
- Command: `SCRIPT`.

## Advantages of Pipelining
- Execute multiple commands in a single round-trip.
- Efficient because reduces round-trip overhead.
- Reduces the number of system calls that our client needs to make.
- allows reading, writing, or a combination of the two at once.
- Command: `client.batch()`:
    - Async/Await pattern is not recommended for use while we're pipelining. Just for the final `exec` command.
    - Returns all the pipelined commands as an array of responses, (like `Promise.all()`

## Advantages of Transactions
- Not guaranteed to execute atomically
- Transaction is like a pipeline that invokes commands atomically. (All commands linked at once), the other ones must wait before executing.
- Command: `client.multi()`
    - As pipelining, the async/await pattern is not recommended for use while we're using transactions. Just for the final `exec` command.

## Pipeline vs Transactions
- Use a pipeline when…
    - You have two or more commands to execute
    - Can wait for the responses of all commands at once.
- Use a transaction if, in addition...
    - You require atomic execution of a set of commands.
    - **You can afford to block other clients while these commands execute.**

We can also create intermediate keys for handling complex `INTERSECT` like operations, but housekeeping is required!
- **Interim sorted sets need to be deleted or expired.**
- **Interim sorted sets need to have unique names to avoid crashes.**

## Streams for feeds
- Allows data syndication for other systems.
- Acts like a global log.
- We can configure a consumer that receives a copy of the data in the feed.
- Is a data structure that models an append-only log. Every Stream has a set of `key:value` pairs. For the Redisolar system, we have these field-value pairs.

Example:
```
- entry:
    - ID: `123456789-0`
    - Fields:
        - `siteId:2`
        - `wHg:0.25`
        - `wHu:0.17`
        - `tempC:18.0`
```
**Graph stream visualization**
![Streams](/streams.png)

## Rate limiters
- Keeps track of user requests.
- Used for protecting server resources
- **Fixed window**
    - Easier to implement
    - Less precise than sliding window
    - Space-efficient (Single redis key)
    - Time-efficient (O(1))
    - One round-trip to Redis.
- **Sliding window**
    - More precise
    - Trickier to implement
    - Uses more space and time

## Blocking commands
- Polling → Each consumer I/O command is executed into Redis.
- Sometimes is better to block a client if the requested value doesn’t exist after polling.
- Prevents unnecessary command round trips associated with polling.

### Supports:
- Lists
- Sorted sets
- Strings
- Pub/Sub features.
- Streams

### Available blocking commands:
- BRPOP: `BRPOP key seconds` Blocking version of RPOP (Removes and returns the last element of a list). Blocks the client for the given seconds, and returns `NIL` if any value is pushed to the list after X seconds.

## Error handling
The captured exceptions have the following structure:

### Single error
Sample:
```
name: ‘ReplyError’
message: ‘ERR wrong number of arguments for ‘set’ command.
command ‘SET’
args [’replyError’]
code ‘ERR’
```

### Pipeline/Transactions errors
Exceptions are not thrown, The catch block is not executed, but it's returned as a part of pipelining EXEC array.

### Connection errors
Returns information about the retry strategy, including delay, attempts, error details, total retry-time, and times connected.
- Redis buffers any executed command and executes it after the connection is recovered and established.
- If we want a “retry strategy” it’s necessary to return the number of milliseconds to wait before trying again in the “retry_strategy” `createClient` Redis callback.

## Performance
- Network latency
- Time complexity
- Atomicity & Blocking

### How to minimize latency?
- Use pipelining when:
    - You’re running more than one command (LOOPS)
    - You don’t need intermediate responses.
- Pipelining reduces:
    - Number of round trips
    - Context switching
    - Syscalls

### Time Complexity
- Be aware of the time complexity of the Redis commands your app uses “Visit Redis.io” website.
- Take care of O(n) Commands.
- Take time to complete (100000000).
- Can use a lot of CPU.
- Might buffer a lot of data.
- Redis is mostly single-threaded.
- Other commands will queue up while a long-running command completes.
- Examples: LRANGE (between 0 to -1) → 100000000 registries.
- **Always avoid blocking commands in production**

**Performance blocking commands visualization**
![Streams](/performance.png)

- The `KEYS` command, or `Client.keys()` should NOT be run in production.
    - Can block for a long time
    - Use SCAN instead.
- Ex: Running `KEYS *` with 4’000.000 keys can take up to 4 seconds to run.

**The administrator can also DISABLE Commands to avoid their use in production.**

### Atomicity and Blocking
- Transactions and Lua scripts block other commands.
- Consider the time complexity of commands run inside transactions and Lua scripts.
- If you don’t need transactional semantics, consider a pipeline (which does not block).

## Debugging
- [MONITOR](https://redis.io/commands/monitor) commands allow to check all the commands sent to Redis.
- Redis includes a proper f[ull-featured debugger](https://redis.io/topics/ldb/) for Lua Scripts.




Earned Certification Link: https://university.redis.com/certificates/761856f316974e0da71b6183250be497
