## DNS Endpoint Pool

Manage and load-balance a pool of service endpoints retrieved from a DNS lookup for a service discovery name.

![Empty pool](https://farm4.staticflickr.com/3480/3192156375_ca9377272a_z.jpg)

## Example

Given a DNS setup like so:

```shell
$ dig +tcp +short SRV my.domain.example.com
0 0 8000 first.example.com.
0 0 8000 second.example.com.
0 0 8000 third.example.com.
```

This library exposes those service endpoints via an interface which will load balance (using unweighted round-robin
only), and apply a circuit breaker to remove failing endpoints from the pool temporarily. Endpoints will be
automatically updated on a timer.

```js
var DNSEndpointPool = require('dns-endpoint-pool');

var pool = new DNSEndpointPool('my.domain.example.com', 10000, 5, 10000);

var endpointInfo = pool.getEndpoint();
if (endpointInfo) {
  sendRequest('http://' + endpointInfo.url, function (err) {
    // let the pool know if the request was successful
    endpointInfo.callback(err);
  });
}
```

If the usage of a endpoint is unsuccessful too many times within a window of time, it will be temporarily disabled.
During this time, `getEndpoints()` will not return that endpoint. Afterwards, it will be returned to the pool for one
single usage, and removed again. If that usage is successful, it is restored to the pool fully. If it is unsuccessful,
it is disabled once again.

## API

### `new DNSEndpointPool(serviceDiscoveryName, ttl, circuitBreakerConfig, onReady)`

Creates a new pool object.

- `serviceDiscoveryName`: the domain name to get endpoint values from
- `ttl`: the time (in ms) that the DNS lookup values are valid for. They will automatically be refreshed on this
  interval.
- `circuitBreakerConfig`: optional configuration for circuit breaker behavior. If specified, errors on a particular endpoint will be tracked and bad endpoints removed from the pool. If none provided, no circuit breaker logic is applied. There are two different circuit-breaker behaviors available:
  - Errors over time: the CB is configured with the an error threshold within a sliding time window. (eg: 5 errors in 10 seconds)
    - `maxFailures`: how many failures from a single endpoint before it is removed from the pool.
    - `failureWindow`: size of the sliding window of time in which the failures are counted.
    - `resetTimeout`: The timeout before a failing endpoint will be re-entered to the pool and tried again.
  - Error rate: the CB is configured with the percentage of errors within a sliding window of requests. (eg: 50% errors over 20 requests).
    - `failureRate`: a number, `0 < n <= 1` that describes the rate at which the endpoint is disabled.
    - `failureRateWindow`: the number of requests over which to calculate the failure rate.
    - `resetTimeout`: The timeout before a failing endpoint will be re-entered to the pool and tried again.
- `onReady`: callback that will be executed after the list of endpoints is fetched for the first time. This does *not* guarantee that the endpoint list is not empty.

### `pool.getEndpoint()`

Returns the next active `endpoint` from the pool, or `null` if none are available. If none are available, the pool will
emit `'noEndpoints'`. If using circuit breakers, you _must_ call `endpoint.callback(err)` with the result of a call to this endpoint.

### `pool.getStatus()`

Returns an object containing information about the health of the pool. There are three values:

- `total`: The total number of endpoints in the pool, in any state.
- `unhealthy`: The number of endpoints which are unavailable (eg: due to their circuit breaker being open)
- `age`: The number of milliseconds since the last successful update of endpoints.

### `endpoint.url`

The endpoint url (without protocol) from the DNS lookup.

### `endpoint.callback(err)`

A callback which should be executed exactly once for each call to `getEndpoint`. If the `err` is truthy, this will count
as a failure of the endpoint. If falsey, it marks the endpoint as successful and allows it to remain in the pool.

## Events

- If a request to update the endpoints fails, the pool will emit an `'updateError'` event. The endpoint pool will continue to
  function, using the previously fetched endpoints.
- If it is not possible to return any values from `getEndpoints()` (because all endpoints are disabled, for example), the
  pool will emit `'noEndpoints'`.


```js
var pool = new DNSEndpointPool('my.domain.example.com', 10000, {
  maxFailures: 5,
  failureWindow: 10000,
  resetTimeout: 10000
});

pool.on('updateError', function (err) {
  log('Could not fetch endpoints');
});
pool.on('noEndpoints', function () {
  log('No endpoints available');
})
```

## Photo credit

[Photo](https://www.flickr.com/photos/perspective23/3192156375) of a scary pool by [perspective23](https://www.flickr.com/photos/perspective23/) used under [CC license](https://creativecommons.org/licenses/by-nc-nd/2.0/)

## License

MIT, SoundCloud.
