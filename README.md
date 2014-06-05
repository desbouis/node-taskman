# node-taskman

[![Build Status](https://travis-ci.org/neoziro/node-taskman.svg?branch=master)](https://travis-ci.org/neoziro/node-taskman)
[![Dependency Status](https://david-dm.org/neoziro/node-taskman.svg?theme=shields.io)](https://david-dm.org/neoziro/node-taskman)
[![devDependency Status](https://david-dm.org/neoziro/node-taskman/dev-status.svg?theme=shields.io)](https://david-dm.org/neoziro/node-taskman#info=devDependencies)

node-taskman is a fast work queue based on redis.

It supports several features:

- worker configuration without restart needed
- take multiple tasks in one time
- unique queue

## Install

```
npm install node-taskman
```

## Usage

````js
var taskman = require('node-taskman');
var driver = new taskman.driver.RedisDriver();
var queue = new taskman.Queue('my-queue', driver);
var worker = new taskman.Worker(queue, driver, 'my-worker', function (data, callback) {
  // process task
  callback();
});

// Start the worker.
worker.start();

// Push a new task.
queue.rpush('some task');
````


### new RedisDriver(options)

Create a redis driver, currently the only driver supported by taskman.

Options avalaible are :

* `port` : Port, default is the redis port, so `6379`.
* `host` : Host, the host, default is `127.0.0.1`.
* `db` : The database, default is 0. The redis driver will perform a `select` in the initialization.
* `queuePrefix` : The queue prefix, default `queue`. If you name your queue "my_queue", the complete name will be "queue:my_queue".
* `workerPrefix` : The worker prefix, default `worker`. If you name your worker "my_worker", the complete name will be "worker:my_worker".

### new Queue(name, driver)

Create a new queue, with a `name` and a `driver`

* `name` : The name of the queue
* `driver` : The driver to use.

````javascript
var taskman = require("node-taskman"), driver, queue;

driver = new taskman.driver.RedisDriver();
queue = new taskman.Queue("my_queue", driver);
````

### queue.rpush(data, callback)

Push a data at the end of the list.

* `data` : The data to push, only string is supported.
* `callback(err, res)` : Called when command is done.

### queue.lpush(data, callback)

Push a data at the beginning of the list.

* `data` : The data to push, only string is supported.
* `callback(err, res)` : Called when command is done.

### queue.rpop(number, callback)

Remove and get the number of element specified at the end of a queue. If you use rpush to add data to the list,
the type is LIFO. We can pass a number to get several data in the queue using pipeling to be more performant.

* `number` : Number of data to get.
* `callback(err, res)` : Called when command is done.

### queue.lpop(number, callback)

Remove and get the number of element specified at the end of a queue. If you use rpush to add data to the list,
the type is FIFO. We can pass a number to get several data in the queue using pipeling to be more performant.

* `number` : Number of data to get.
* `callback(err, res)` : Called when command is done.

### queue.llen(callback)

Count the number of element in the queue.

* `callback(err, res)` : Called when command is done.

### new Worker(queue, driver, name, action, options)

Create a worker to take items of a queue and execute an action.

* `queue` : The `Queue` that will be processed.
* `driver` : The driver to use to stock informations of the worker.
* `name` : The name of the worker, to be identified.
* `action(data, callback, worker)` : A function called when a data is pop from the queue
   * `data` : An array of values get from the queue, the number of values is the same, but empty values are filled by null.
   * `callback` : The callback to call to trigger the next tick of the worker.
   * `worker` : The current worker instance.
* `options` : Options
   * `type` : FIFO or LIFO
   * `loopSleepTime` : The time in seconds to sleep between two tick, default `0`.
   * `pauseSleepTime` : The time in seconds to sleep between each tick in pause mode, default `5`.
   * `waitingTimeout` : The time in seconds to wait when the queue is empty, default `1`.
   * `dataPerTick` : The number of item popped from the queue, default `1`.
   * `expire` : Expire time in seconds, default `17280000` (200 days).
   * `unique` : A key can't be duplicated in queue, default `false`.
   * `timeout` : A timeout for the pop's action, default `300000` (5 minutes).

### worker.start(callback)

Start the worker, execute the `callback` when it's started.

* `callback(err, res)` : Called when the command is done.

### worker.stop(callback)

Stop the worker, execute the `callback` when it's stopped.

* `callback(err, res)` : Called when the command is done.

### worker.pause(callback)

Pause the worker, execute the `callback` when it's paused.

* `callback(err, res)` : Called when the command is done.

### worker.setEndDate(date, callback)

Set an end date to the worker, when the end date is reached, the worker die.

* `date` : Date at json format (`new Date().toJSON()`).
* `callback(err, res)` : Called when the command is done.

### worker.setLoopSleepTime(time, callback)

Change the loop sleep time.

* `time` : Time in seconds.
* `callback(err, res)` : Called when the command is done.

### worker.setPauseSleepTime(time, callback)

Change the pause sleep time.

* `time` : Time in seconds.
* `callback(err, res)` : Called when the command is done.

### worker.setDataPerTick(number, callback)

Change the number of items returned in each tick.

* `number` : Number of items per tick.
* `callback(err, res)` : Called when the command is done.

### worker.setWaitingTimeout(timeout, callback)

Change the waiting timeout of the worker.

* `timeout` : Timeout in second.
* `callback(err, res)` : Called when the command is done.

### worker.setType(type, callback)

Change the type of the worker.

* `type` : Type of the process, LIFO or FIFO.
* `callback(err, res)` : Called when the command is done.

### worker.getCompleteName()

Get the complete name of the worker.

`return` a string that is the complete name of the worker.

### worker.getInfos(callback)

Get all infos of the worker.

* `callback(err, res)` : Called when the command is done.

## In redis database

Every information of the worker are avalaible in the redis database, an example:

````js
var taskman = require("node-taskman"), driver, queue, worker;

driver = new taskman.driver.RedisDriver();
queue = new taskman.Queue("test_queue", driver);

worker = new taskman.Worker(queue, driver, "test_worker", function(data, callback){
   console.log(data);
   callback();
});

queue.rpush("Hello World", function(){
   worker.start();
});
````

When this script is executed, we can connect redis and execute some commands:

````
redis-cli
redis 127.0.0.1:6379> hgetall worker:test_queue:test_worker
 1) "waiting_timeout"
 2) "1"
 3) "loop_sleep"
 4) "0"
 5) "pause_sleep_time"
 6) "5"
 7) "data_per_tick"
 8) "1"
 9) "action"
10) "function (data, callback){\n   console.log(data);\n   callback();\n}"
11) "type"
12) "FIFO"
13) "language"
14) "nodejs"
15) "process_id"
16) "23985"
17) "status"
18) "WAITING"
19) "status_changedate"
20) "2012-06-05T09:27:02.693Z"
21) "cpt_action"
22) "1"
23) "start_date"
24) "2012-06-05T09:26:42.653Z"
25) "end_date"
26) "2012-06-05T14:14:42.654Z"
````

And we can see some informations:

* `language` : The language that execute the worker
* `process_id` : The PID of the worker
* `status` : The current status of the worker
* `status_changedate` : The last move
* `cpt_action` : The number of tick

We have our parameters too, and we can change them:

````
redis 127.0.0.1:6379> hset worker:test_queue:test_worker loop_sleep 20
(integer) 0
````

So in real time you can control your worker!

## License

MIT

## Credits

Written and maintained by [Greg Bergé][neoziro] and [Justin Bourrousse][JBustin]

An original idea by [David Desbouis][desbouis].

Build an used on [Le Monde.fr](http://www.lemonde.fr).

[neoziro]: http://github.com/neoziro
[desbouis]: http://github.com/desbouis
[JBustin]: http://github.com/JBustin
