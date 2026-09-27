# taskpace

Taskpace is a task scheduler and rate limiter for Node.js and the browser. It queues work and runs it under limits you set: how many jobs may run at once, how long to wait between starts, and how large the waiting line may grow.

It has no runtime dependencies. The same limiter can also share its counters across processes through Redis.

## Install

```bash
npm install taskpace
```

```js
const Taskpace = require("taskpace");
```

```js
import Taskpace from "taskpace";
```

Browsers and Node versions older than 6 should load the ES5 build:

```js
const Taskpace = require("taskpace/es5");
```

## Create a limiter

```js
const limiter = new Taskpace({
  maxConcurrent: 2,
  minTime: 200
});
```

`minTime` is the pause after one job starts before another may start. `maxConcurrent` is how many jobs may be in progress together. Either option can be used alone. Together they keep a steady pace instead of a burst followed by a long gap.

Pass `null` for `maxConcurrent` when you do not want a concurrency cap.

### Run a promise

```js
const page = await limiter.schedule(() => fetch(url).then((res) => res.text()));
```

`schedule` accepts the function and any arguments that function should receive:

```js
const body = await limiter.schedule(fetchText, url, { timeout: 5000 });
```

### Wrap an existing function

```js
const fetchText = limiter.wrap(loadPage);

const body = await fetchText("/reports");
```

The wrapped function has the same arguments as the original. It returns a promise that settles when the job finishes, or when Taskpace itself rejects the job.

### Callbacks

```js
limiter.submit(readFile, path, (err, data) => {
  if (err) return console.error(err);
  console.log(data);
});
```

If you do not need the callback, pass `null`. The function you submit still has to call its own callback so the limiter knows the job is finished. A job that never calls back occupies a slot until you set an `expiration`.

## Limiter options

| Option | Default | What it does |
| --- | --- | --- |
| `maxConcurrent` | `null` | Maximum number of jobs in progress. `null` means no cap. |
| `minTime` | `0` | Milliseconds to wait after a job starts before the next one can start. |
| `highWater` | `null` | Maximum queue length. Past this point, the chosen `strategy` drops work. Ignored when `null`. |
| `strategy` | `Taskpace.strategy.LEAK` | What to do when the queue would pass `highWater`. |
| `penalty` | `15 * minTime`, or `5000` when `minTime` is `0` | How long `BLOCK` stays closed after the queue overflows. |
| `reservoir` | `null` | How many jobs may start before the limiter pauses. `0` stops new starts until the value rises again. Jobs can still be queued. |
| `reservoirRefreshInterval` | `null` | How often, in milliseconds, to set `reservoir` back to `reservoirRefreshAmount`. |
| `reservoirRefreshAmount` | `null` | Value assigned to `reservoir` on each refresh. |
| `reservoirIncreaseInterval` | `null` | How often, in milliseconds, to add `reservoirIncreaseAmount` to `reservoir`. |
| `reservoirIncreaseAmount` | `null` | Amount added on each increase. |
| `reservoirIncreaseMaximum` | `null` | Ceiling for `reservoir` while increases are enabled. |
| `trackDoneStatus` | `false` | Keep finished jobs visible to `counts()` and `jobStatus()`. Uses more memory. |
| `Promise` | the built-in `Promise` | Promise implementation Taskpace should use. |

Refresh and increase intervals must be a multiple of 250. When the limiter stores state in Redis, use a multiple of 5000.

The interval clock starts when the limiter is created. A refresh can land immediately after you enqueue a burst, so the limiter may start two allotments back to back. Keep `minTime` or `maxConcurrent` set if that burst would be a problem.

A reservoir interval holds a timer, so the limiter will not be garbage-collected until you call `disconnect()`. The process can still exit without that call.

## Allowances that refill

Use a reservoir when a quota resets on a clock, rather than when you only need a smooth gap between calls.

This allows 30 starts, then restores that allowance every minute:

```js
const limiter = new Taskpace({
  reservoir: 30,
  reservoirRefreshAmount: 30,
  reservoirRefreshInterval: 60 * 1000,
  maxConcurrent: 3,
  minTime: 100
});
```

This starts with 20 starts and adds 1 every second, never going above 20:

```js
const limiter = new Taskpace({
  reservoir: 20,
  reservoirIncreaseAmount: 1,
  reservoirIncreaseInterval: 1000,
  reservoirIncreaseMaximum: 20,
  maxConcurrent: 4,
  minTime: 50
});
```

When a refresh fires, every newly allowed job becomes eligible at once. `minTime` and `maxConcurrent` are what spread those starts out.

```js
const left = await limiter.currentReservoir();
const updated = await limiter.incrementReservoir(5);
```

## Options on a single job

Pass an options object as the first argument to `schedule` or `submit`. On a wrapped function, call `withOptions`.

```js
await limiter.schedule({ priority: 1, id: "invoice-9" }, sendInvoice, invoice);

limiter.submit({ weight: 2 }, writeRows, rows, done);

const send = limiter.wrap(postJson);
await send.withOptions({ expiration: 8000 }, "/charges", payload);
```

| Option | Default | What it does |
| --- | --- | --- |
| `priority` | `5` | Integer from `0` (runs sooner) to `9` (runs later). Priorities only order work that is actually waiting, so set `maxConcurrent`. |
| `weight` | `1` | How much this job counts against `maxConcurrent` and `reservoir`. Integer `0` or greater. |
| `expiration` | `null` | Milliseconds the job may run. After that, it fails with a `TaskpaceError`. |
| `id` | `"<no-id>"` | Label used in debug output and by `jobStatus()`. |

## When the queue is too long

If adding a job would make the queue longer than `highWater`, Taskpace applies `strategy`. Nothing in this section runs while `highWater` is `null`.

| Strategy | Behavior |
| --- | --- |
| `Taskpace.strategy.LEAK` | Drop the oldest waiting job that has the lowest priority. If every waiting job outranks the new one, the new job is the one dropped. |
| `Taskpace.strategy.OVERFLOW_PRIORITY` | Drop a waiting job only when it is less important than the new one. Otherwise reject the new job. |
| `Taskpace.strategy.OVERFLOW` | Reject the new job. Priority is ignored. |
| `Taskpace.strategy.BLOCK` | Drop the queue and refuse new jobs until `penalty` milliseconds pass with no new arrival. Priority is ignored. One limiter entering this state blocks every limiter that shares its Redis datastore. |

Listen for drops:

```js
limiter.on("dropped", (job) => {
  console.log("dropped", job.options.id);
});
```

## What the function you pass must do

`schedule` and `wrap` consider the job finished when the returned promise settles. Return only after the work is done:

```js
await limiter.schedule(() => {
  return Promise.all(items.map((item) => save(item)));
});
```

If you pass an object method, keep its `this` binding:

```js
limiter.schedule(() => client.query(sql));
```

Listen for `"error"`. Those events are how listener exceptions and Redis failures surface. They are not delivered through the job promise.

`submit` always needs a callback argument, even if that argument is `null`. The task itself must still invoke its callback.

## Job stages

1. **Received.** The job has entered the limiter and has not yet been accepted into the queue.
2. **Queued.** It is accepted, and its start time still depends on earlier jobs.
3. **Running.** It has left the queue and is waiting out the computed `minTime` delay.
4. **Executing.** Its function is running.
5. **Done.** It has finished.

Finished jobs are not retained unless the limiter was created with `trackDoneStatus: true`.

```js
limiter.counts();
// { RECEIVED, QUEUED, RUNNING, EXECUTING, DONE }

limiter.jobStatus("invoice-9"); // "QUEUED", or null if that id is unknown
limiter.jobs("EXECUTING");      // ["invoice-9"]
limiter.jobs();                  // every known id

limiter.queued();                // waiting jobs in this limiter
limiter.queued(1);               // waiting jobs at priority 1
limiter.empty();                 // true when nothing is RECEIVED or QUEUED here

await limiter.clusterQueued();   // waiting jobs across limiters that share state
await limiter.running();         // total weight of RUNNING and EXECUTING work in the shared state
await limiter.done();            // total weight of DONE work in the shared state
await limiter.check();           // true if a new job would start immediately
```

`running()` and `done()` report weight, not a plain job count. `done()` does not require `trackDoneStatus`.

## Events

```js
limiter.on("error", (err) => {});
limiter.on("failed", (err, info) => {});
limiter.on("retry", (message, info) => {});
limiter.on("empty", () => {});
limiter.on("idle", () => {});
limiter.on("dropped", (dropped) => {});
limiter.on("depleted", (isEmpty) => {});
limiter.on("debug", (message, data) => {});
limiter.on("received", (info) => {});
limiter.on("queued", (info) => {});
limiter.on("scheduled", (info) => {});
limiter.on("executing", (info) => {});
limiter.on("done", (info) => {});
```

`"empty"` fires when this limiter has nothing received or queued. `"idle"` fires when that is true and `running()` is `0`. `"depleted"` fires when `reservoir` hits `0`; the argument is the current result of `empty()`. `"dropped"` fires when a strategy discards a job.

Stage events (`received`, `queued`, `scheduled`, `executing`, `done`) describe jobs on this limiter only.

`once` listens a single time. `removeAllListeners` clears listeners, optionally for one event name.

## Retry a failed job

Return a delay in milliseconds from the `"failed"` handler. Return `0` to retry immediately. Return nothing to let the failure stand.

```js
limiter.on("failed", (err, info) => {
  if (info.retryCount < 2) return 100;
});

limiter.on("retry", (message, info) => {
  console.log("retrying", info.options.id);
});
```

A job that is waiting to retry stays in `EXECUTING`. It still counts toward `maxConcurrent`, and the delay does not consult `minTime`.

## Update, stop, and chain

```js
await limiter.updateSettings({ minTime: 400, maxConcurrent: 1 });
```

The object matches the constructor options. Jobs that are already scheduled keep the previous timing.

```js
await limiter.stop({
  dropWaitingJobs: true,
  dropErrorMessage: "This limiter has been stopped.",
  enqueueErrorMessage: "This limiter has been stopped and cannot accept new jobs."
});
```

`stop` rejects further arrivals. With `dropWaitingJobs: true` (the default), jobs that are received, queued, or running are failed with `dropErrorMessage`. Jobs already executing are allowed to finish. Set `dropWaitingJobs` to `false` to wait for the rest of the queue as well.

`chain` sends a job to another limiter once this one is ready to run it. Both limits apply:

```js
const reports = new Taskpace({ minTime: 50 });
const uploads = new Taskpace({ maxConcurrent: 1 });
const globalCap = new Taskpace({ maxConcurrent: 4 });

reports.chain(globalCap);
uploads.chain(globalCap);

reports.chain(null); // remove the link
```

## One limiter per key

`Taskpace.Group` builds a limiter the first time you ask for a key, and reuses it after that.

```js
const group = new Taskpace.Group({ maxConcurrent: 1, minTime: 100 });

group.on("created", (limiter, key) => {
  limiter.on("error", (err) => console.error(key, err));
});

await group.key(userId).schedule(() => charge(userId));
```

The handler for `"created"` runs before `key` returns the new limiter. Options passed to the group are copied onto each limiter it creates.

Limiters that stay idle are removed after 5 minutes. Pass `timeout` in milliseconds to change that.

```js
group.updateSettings({ minTime: 250 }); // applies to limiters created afterward
group.keys();
group.limiters(); // [{ key, limiter }, ...]
await group.deleteKey(userId);
await group.clusterKeys(); // keys for this group id across Redis
```

In Redis mode, a generated limiter id looks like `${group.id}-${key}`.

## Group calls into batches

`Taskpace.Batcher` collects values and flushes them together. It does not apply a rate limit.

```js
const batcher = new Taskpace.Batcher({
  maxTime: 200,
  maxSize: 25
});

batcher.on("batch", async (rows) => {
  await insertMany(rows);
});

await batcher.add(row);
```

| Option | Default | What it does |
| --- | --- | --- |
| `maxTime` | `null` | Longest a value may wait before a flush. |
| `maxSize` | `null` | Largest batch. A flush also happens when this size is reached. |

`add` resolves when that value has been handed to a `"batch"` listener.

## Share state with Redis

Set `datastore` to `"redis"` or `"ioredis"` so every limiter with the same `id` shares counters. Install the client yourself (`redis` or `ioredis`). It is not bundled.

```js
const limiter = new Taskpace({
  id: "checkout",
  maxConcurrent: 5,
  minTime: 100,
  datastore: "ioredis",
  clearDatastore: false,
  clientOptions: {
    host: "127.0.0.1",
    port: 6379
  }
});
```

| Option | Default | What it does |
| --- | --- | --- |
| `datastore` | `"local"` | `"local"`, `"redis"`, or `"ioredis"`. |
| `clearDatastore` | `false` | On startup, delete stored state for this `id`, including saved settings. |
| `clientOptions` | `{}` | Passed through to the Redis client. |
| `clusterNodes` | `null` | ioredis only. When set, the client is `new Redis.Cluster(clusterNodes, clientOptions)`. |
| `timeout` | `null` | Remove this limiter's Redis keys after this many milliseconds without activity. Groups default this to `300000`. |
| `Redis` | `null` | The `redis` or `ioredis` module, if Taskpace should not load it itself. |
| `connection` | `null` | An existing `Taskpace.RedisConnection` or `Taskpace.IORedisConnection`. When set, `datastore`, `clientOptions`, and `clusterNodes` are ignored. |

The first limiter to connect stores its constructor options. Later limiters with that `id` use the stored options. Call `updateSettings` to change them for everyone. `clearDatastore: true` wipes that record first.

Give every limiter and group an `id`. It is part of the Redis key names. Also set `expiration` on jobs. If a process dies mid-job, the shared running count is released when the expiration elapses.

Queues stay in the process that called `schedule` or `submit`. They are not written to Redis, and they disappear if that process exits. Because of that:

- Priority and arrival order are guaranteed only among jobs on the same limiter.
- `highWater` is local, except that `BLOCK` freezes the whole shared limiter.
- `"empty"` refers to the local queue. `"idle"` also requires that nothing is running anywhere in the shared state.

`minTime` does not subtract network delay. Taskpace makes one Redis round trip per stage change. Keep the server close, and keep clocks aligned across hosts.

Redis Cluster needs `datastore: "ioredis"` and `clusterNodes`. Redis Sentinel also needs `ioredis`.

If a bundler cannot see the optional client import, pass the module in:

```js
import Redis from "ioredis";

const limiter = new Taskpace({
  id: "checkout",
  datastore: "ioredis",
  clientOptions: { host: "127.0.0.1", port: 6379 },
  Redis
});
```

### Connection helpers

You can call these with `datastore: "local"` as well, so calling code does not have to branch.

```js
limiter.on("error", (err) => console.error(err));

await limiter.ready();

limiter.on("message", (msg) => console.log(msg));
await limiter.publish("reload");

const { client, subscriber } = limiter.clients();
```

`ready` resolves after Redis is connected. Calls made earlier are held until then. Connection failures arrive on `"error"`.

`publish` sends a string to every limiter on that shared state. Stringify objects yourself.

### One connection for many limiters

Taskpace opens two clients: one for commands and one for subscriptions. Build a connection once and pass it in to share them.

```js
const connection = new Taskpace.IORedisConnection({
  clientOptions: { host: "127.0.0.1", port: 6379 }
});

connection.on("error", (err) => console.error(err));

const limiter = new Taskpace({ id: "checkout", connection });
const group = new Taskpace.Group({ connection });
```

`Taskpace.RedisConnection` is the matching helper for the `redis` package. You can also hand over a client you already opened. Taskpace still creates the subscriber client.

```js
const connection = new Taskpace.RedisConnection({ client });
```

`disconnect()` closes clients that Taskpace opened. If you created the connection yourself, call `connection.disconnect()` instead.

Taskpace stores its Redis data under its own keys and channels, separate from other application data. It loads a few Lua scripts with `SCRIPT LOAD`. `SCRIPT FLUSH` removes those scripts and breaks connected limiters until another one loads them again.

## Failures from Taskpace

When Taskpace rejects a job, the error is a `Taskpace.TaskpaceError`. That includes dropped jobs, expirations, and jobs refused after `stop()`.

```js
try {
  await limiter.schedule(() => callApi());
} catch (err) {
  if (err instanceof Taskpace.TaskpaceError) {
    console.error(err.message);
  } else {
    throw err;
  }
}
```

`"debug"` prints the decisions the limiter is making. Job `id` values make that log easier to follow.

## Development

Source lives in `src/`. Build it, then run the tests:

```bash
./scripts/build.sh
npm test
```

`./scripts/build.sh dev` is a faster local compile. Run a full build before you rely on the ES5 bundle or the type declarations. Clustering tests need a local Redis server. Host and port overrides go in `.env`, and `npm run test-all` runs that suite.

## License

MIT. See [LICENSE](LICENSE).
