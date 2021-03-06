# qb [![Build Status][1]][2]

A super-simple service-provder framework.

`qb` makes it easy to create a service-oriented architecture built on Redis queues (specifically [relyq](https://github.com/Rafflecopter/relyq)).

To get started, check out the [example app](https://github.com/Rafflecopter/node-qb/blob/master/example/app.js) look below.

Minimal Example:

```javascript
require('qb').init({prefix: 'qb'})
  .can('email', sendEmail)
  .speaks('http', {port: 8000})
  .start();
```

Full Example:

```javascript
// Init
var qbPkg = require('qb'),
  mdw = qbPkg.mdw;

// Initialize qb
var qb = qb.init({
    prefix: 'my-emailer'
  })

  // Service to provide
  .can('email', [max_concurrent_callbacks,] sendEmail )
  .can('add-subscriber', [max_concurrent_callbacks,] addSubscriber)

  // Tell it what dialects to speak (i.e. communicate with other qb instances on)
  .speaks('http', { port: 8000, base: '/qb' })
  .speaks('messageq', { discovery_prefix: 'qb:discovery' })

  // Add some middleware
  .pre('push').use(mdw.setTimestamp('received'))
  .pre('process').use(mdw.setTimestamp('processed'))
  .post('fail').use(sendFailureReport /* custom middleware */);

  // Start!
  .start();

// Contact another qb instance

// Option 1: use the returned dialect
var myFriend = qb.contacts('http://my.friend.com/qb');
myFriend.push('other-service', {a:'new', job:'description'});

// Option 2: use an alias
qb.contacts('messageq://some-channel', 'somechan');
qb.contacts('messageq://other-channel', 'otherchan');

// Option 3: Always use .contact(uri)
qb.contact('http://my.service.com/api/do/something').push(task);

// ...later...
qb.contact('somechan')
  .subscribe(function(msg) { msg.email = 'abc@jon.com' }, 'email')
  .publish({do:'this', a:'lot'});

qb.contact('otherchan').publish({seri:'ously'});

// Some time later, graceful shutdown
qb.end(onend);

// Handle errors
qb.on('error', function (err) {
  var args = Array.prototype.slice.call(arguments, 1);
  //etc..
})

// You can locally push tasks
qb.push('email', {email: 'this-guy', template: 'whatever'})

  // Or use deferred tasks
  .push('email', {when: Date.now() + 60000, email: 'another-guy', template: 'whatever'})

  // And you can do recurring tasks
  .push('email', {every: 60*60*1000, email: 'our-man', template: 'recurring'});
```

## Features

- Easy service setup: `.can` method
- Multiple dialects for sending and receiving tasks: `.speaks` and `.dialect().push` methods
- Flexible middleware system: `.pre` and `.post` methods
- Graceful shutdown: `.post('end', onReadyToShutdown).end` methods
- Easy configuration

## Install

```
npm install qb
```

## Options

`qb` makes a lot of assumptions on how you might use it. But most of those are configurable. I've highlighted the most commonly used options, while being complete with the description of all options.

### on `.init()`

The `require('qb').init()` function takes an options object. The options available are below:

```javascript
require('qb').init();
//or
require('qb').init(options);
```

Common:

- `prefix: 'my-services'` (default: 'qb') The [relyq](https://github.com/Rafflecopter/relyq) service prefix for Redis queue keys. (Service endpoints will take the type name after the prefix.) This should be unique for each type of service. So two qb instances that share the same prefix will share the same queues, which is good for redundancy but bad for different instances

Others:

- `clean_finish: true` (default: true) If true, no jobs are kept after being successfully finished.
- `delimeter: ':'` (default: ':') Sets the Redis delimeter
- `idfield: 'id'` (default: 'id') Sets the id field to use on tasks. IDs will be distinct and uniquely set on push if they don't already exist.
- `Q: relyq.RedisJsonQ` (defaults to RedisJsonQ) A [relyq](https://github.com/Rafflecopter/relyq) queue type to use. The suggested ones are `RedisJsonQ`, `RedisMsgPackQ`, and `MongoQ` (which only uses mongo for storing task objects, not the queue itself which is still in Redis).
  - If using `relyq.MongoQ`, additional options are required: `mongo: mongodb.mongoClient`, `db: dbname`, and `collection: collname`.
- `max_concurrent_callbacks: 100` (defaults to 100) Set the default max_concurrent_callbacks in case its not passed in on `.can`.
- `allow_defer: true` (defaults to true) Allows deferred tasks
  - `defer_field: 'when'` (defaults to 'when') Notes a field, that if filled and allow_defer is on, will create a deferred job deferred until the timestamp in the defer_field (which should be a javascript timestamp in milliseconds since 1970).
  - `defer_polling_interval: 1000` (in ms, defaults to 1s) Polling interval for deferred tasks to be pulled from the database. There is no blocking call so polling is our best choice.
- `allow_recur: true` (defaults to true) Allows recurring tasks
  - `recur_field: 'when'` (defaults to 'when') Notes a field, that if filled and allow_recur is on, will create a recurring job recurring every `task[recur_field]` (in ms).
  - `recur_polling_interval: 60000` (in ms, defaults to 60s) Polling interval for recurring tasks to be pulled from the database. There is no blocking call so polling is our best choice.
- `end_timeout: 10000` (defaults to 10s) Timeout on `.end(cb)` calls
  - `catch_sigterm_end: true` (defaults to true) Tells qb to catch sigterm signals and perform an `.end()` call.

## Notes

- Recurring tasks must include an ID so that they are not duplicated. The ID field defaults to 'id'.
- `qb.log` is the singleton `require('book')` instance from the [node-book](https://github.com/shtylman/node-book) package. You can configure the singleton and all `qb.log` calls will be formatted as appropriate.

## Dialects

Each dialect has its own specific options, which are passed in upon doing `speaks(dialect, options)`.

### http

The http dialect is simple.

```javascript
qb.speaks('http', { port: 8000, base: '/qb-api' })
  .start()
  .speak('http')
    .to('http://some.other.server.com/qb')
      .push('service-name', {task:'task',desc:'ription'});
```

The `.speaks.start` starts up a simple express server, while the `.speak.to.push` uses request to communicate with another qb instance.

Options:

- `port` Port Number (if not present, no server will be started to listen)
- `app` Allows one to pass in an express app of their choosing.
- `base` Base api prefix
- `retry` Number of retries before quitting a push call.

### messageq

[messageq](https://github.com/Rafflecopter/node-messageq) is a simple, reliable Redis-backed task queue based pub/sub messaging system based on [relyq](https://github.com/Rafflecopter/relyq). It is based on a pub/sub model and an example follows.

Options:

- `discovery_prefix: 'my-soa-discovery'` (required) - Redis key prefix for the discovery service. This must be same across all qb instances that want to talk to each other.
- `ttl: 5000` (default: 5s (in ms)) - Time to live between cached subscriber requests. Use a large one for long-term subscribers.

Example:

```javascript
qb.speaks('messageq', {discovery_prefix: 'qb:discovery'})
  .start()
  .speak('messageq')
    .to('some-channel')
      .subscribe(functionToRun, 'service-to-trigger')
    .to('other-channel')
      .subscribe('service-to-trigger', 'another-service-to-trigger')

      .publish({a:'message',on:'the-channel'});
```

### nats

A pub/sub dialect based on the [nats](https://github.com/derekcollison/nats) server using the [node.js implementation](https://github.com/derekcollison/node_nats). The options passed on `options.nats` or as `{nats: {}}` in are the connection options for the nats `.connect()` function.

_Note_: I suggest running the [gnatsd](https://github.com/apcera/gnatsd) server which is written in Go. I like Go better than ruby (also its probably faster).

### Implementing Dialects

There are two types of dialects, RPC (remote procedure call) which has a `.push` method to push a task onto another qb instance and pub/sub types which have `.subscribe` and `.publish` methods to which communicate on their own channels and do things upon receiving a message on a subscribed channel (including pushing the message as a task onto a work queue).

See `lib/dialect_implementations/http.js` for an RPC example and `lib/dialect_implementations/messageq.js` for a pub/sub example.

_Note_: QB doesn't actually allow RPC, only the use of it to push tasks onto another QB instance.

## Middleware

Middleware can be put on the following events: `push`, `process`, `fail`, `finish`, and `error`. All middleware can be either `.pre(event)`, `.post(event)`, or `.on(event)` to control flow.

These are the primary points for middleware:

- `.pre('push').use(function (type, task, next) {})` Before a push occurs, modify the task. Erroring here will cause an error response by service-oriented dialects such as api `.push` and `http` dialect.
- `.on('finish').use(function (type, task, next) {})` Do something upon a successful completion of a task. If `clean_finish` is `true` (by default it is), this is the last point which the task will be available before being dropped.
- `.on('fail').use(function (type, task, next) {})` Do something upon a failed task. `task.error` will be available as a stringified error form.
- `.on('error').use(function (err, next) {})` Do something upon an internal error occurring.

Provided Middleware

```javascript
qb.pre('push')
  .use(qb.mdw.setTimestamp('field-to-set-in-the-task'))

qb.on('fail')
  .use(qb.mdw.retry(['service-to-retry','another-service'], 2 /* times */))
```

## Tests

```
npm install -g nodeunit
npm test
```

## TODO

- logging using [book](http://npmjs.org/book)
- Html gui akin to [kue](http://npmjs.org/kue)

## License

See LICENSE file.

[1]: https://travis-ci.org/Rafflecopter/node-qb.png?branch=master
[2]: http://travis-ci.org/Rafflecopter/node-qb