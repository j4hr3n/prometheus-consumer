# Prometheus metrics consumer

A prometheus based streaming consumer for metric streams.

## How it works

This module understands metric streams generated by @metrics/client and makes
opinionated (but overridable) decisions about how to create prometheus metrics from
them.

The [prom-client](https://github.com/siimon/prom-client) module is required to gather the metrics. It must
be passed in as a constructor option (see below).

Additionally, Prometheus expects that your app serve a metrics scraping page.
Refer to the [prom-client](https://github.com/siimon/prom-client) module for details.

For apps using express, this would look like:

```js
const { register } = require('prom-client');

app.get('/_/metrics', (req, res) => {
    res.set('Content-Type', register.contentType).end(register.metrics());
});
```

**An Example.**

Given a metric with a time value:

```js
{
    name: 'my_metric_with_time',
    description: 'Metric that measures time',
    time: 1231432423
}
```

`@metrics/prometheus-consumer` will either create a new prometheus Histogram for
`my_metric_with_time` or update one if it already exists.

## Usage

### Step 1.

Create a new instance of `@metrics/prometheus-consumer` to be our metrics consumer.

```js
const promClient = require('prom-client');
const metricsConsumer = new PrometheusConsumer({ client: promClient });
```

### Step 2.

Pipe the metrics output stream directly into our consumer making sure to add an error handler to avoid uncaught exceptions.

```js
const client = new MetricsClient();

metricsConsumer.on('error', err => console.error(err));

client.pipe(metricsConsumer);
```

### Step 3.

Render a metrics page for prometheus to scrape.
In this example we use express (though any framework will work) to render out metrics on the route `/metrics`.

```js
app.get('/metrics', (req, res) => {
    res.set('Content-Type', metricsConsumer.contentType()).send(
        metricsConsumer.metrics(),
    );
});
```

## API

-   [constructor](#constructoroptions)
    -   [override](#overridename-config)
    -   [metrics](#metrics)
    -   [contentType](#contenttype)

### constructor(options)

Create a new metrics consumer instance ready to have metrics piped into it.

_Examples_

```js
const consumer = new PrometheusConsumer({ client: promClient });
```

remember to add an error handler to the consumer to avoid uncaught exception errors.

```js
consumer.on('error', err => console.error(err));
```

#### options

| name                                  | description                                                                                | type     | default |
| ------------------------------------- | ------------------------------------------------------------------------------------------ | -------- | ------- |
| [client](#client)                     | Prom client module dependency.                                                             |
| [bucketStepStart](#bucketstepstart)   | Value to start bucket generation from. Each step increases from here by `bucketStepFactor` | `number` | 0.001   |
| [bucketStepFactor](#bucketstepfactor) | Scaling factor for bucket creation. Must be > 1                                            | `number` | 1.15    |
| [bucketStepCount](#bucketstepcount)   | Number of times `bucketStepFactor` should be applied to `bucketStepStart`                  | `number` | 45      |

##### client

Prom client module dependency. Passed in this way in order to avoid having a hard dependency on a specific version of prom-client.

_Example_

```js
const consumer = new PrometheusConsumer({ client: promClient });
```

##### bucketStepStart

Value to start bucket generation from. Each step increases from here by `bucketStepFactor`

_Example_

```js
const consumer = new PrometheusConsumer({ bucketStepStart: 1 });
```

##### bucketStepFactor

Scaling factor for bucket creation. Must be > 1

_Example_

```js
const consumer = new PrometheusConsumer({ bucketStepFactor: 2 });
```

##### bucketStepCount

Number of times `bucketStepFactor` should be applied to `bucketStepStart`

_Example_

```js
const consumer = new PrometheusConsumer({ bucketStepCount: 8 });
```

### .override(name, config)

Override handling of a specific metric (by name). This is useful if the defaults
do not produce the desired result for a given metric. You can change what type
of prometheus metrics are generated by setting `type` and for histograms, you
can override bucket handling.

#### name

An alpha-numeric (plus the underscore character) string name of metric to be
overriden. Any metrics that match this name will be processed using the override
config (see below)

#### config

| name    | description                                                                 | type       | default   |
| ------- | --------------------------------------------------------------------------- | ---------- | --------- |
| type    | One of `histogram`, `counter`, `gauge` or `summary`                         | `string`   |           |
| labels  | Array with allowed values: `url`, `method`, `status`, `layout` and `podlet` | `string[]` |           |
| buckets | An object, see `config.buckets` below                                       | `object`   | see below |

##### buckets

| name             | description                                                                                | type     | default |
| ---------------- | ------------------------------------------------------------------------------------------ | -------- | ------- |
| bucketStepStart  | Value to start bucket generation from. Each step increases from here by `bucketStepFactor` | `number` | 0.001   |
| bucketStepFactor | Scaling factor for bucket creation. Must be > 1                                            | `number` | 1.15    |
| bucketStepCount  | Number of times `bucketStepFactor` should be applied to `bucketStepStart`                  | `number` | 45      |

_Example_

Override a specific time based metric to only be handled as a counter.

```js
consumer.override('my_time_based_metric', { type: 'counter' });
```

_Example_

Override labels for a metric.

```js
consumer.override('my_time_based_metric', {
    labels: ['url', 'method'],
});
```

_Example_

Override buckets for a specific time based metric.

```js
consumer.override('my_time_based_metric', {
    buckets: {
        bucketStepStart: 1,
        bucketStepFactor: 2,
        bucketStepCount: 8,
    },
});
```

### .metrics()

Returns the generated metrics text ready for scraping by prometheus. This should be used in conjunction with [.contentType()](#contenttype)

_Example_

```js
app.get('/metrics', (req, res) => {
    res.set('Content-Type', metricsConsumer.contentType()).send(
        metricsConsumer.metrics(),
    );
});
```

### .contentType()

Returns the correct content type to use when rendering metrics text for scraping by prometheus. See [.metrics()](#metrics) for an express js usage example.
