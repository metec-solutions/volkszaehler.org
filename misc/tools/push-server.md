# push-server

The `push-server` is a volkszaehler component that provides realtime updates of measurements to connected clients.

While the classic volkszaehler web frontend originally used polling to update its charts, by enabling `push-server` it will update whenever a new measurement is available.


## Protocols

`push-server` allows vzlogger and volkszaehler middleware to push data to clients whenever it changes, instead of relying on clients to poll for updates.

Two main protocols are implemented:

  - WAMP: used by the volkszaehler frontend (via autobahn.js library). Frontend uses WAMP to subscribe to updates for those channels that are currently selected.
  - Plain web sockets: while WAMP is highly usable for browser communication the protocol incurrs some overhead and implementations are not available for every single platform. Plain web sockets close this gap and can be used to connect additional, non-frontend clients.


### Additional protocols

Some users have expressed need for additional protocols, especially MQTT. Those can be implemented with the flexibilit given be `push-server`.

The suggested approach is to deploy a (small) integration component like e.g. NodeJS-based `node-red` and use plain web sockets to communicate with `push-server`. Any additional protocol can be implemented on top of `node-red`.


## Architecture

Intent of the `push-server` is to provide realtime updates for clients using an easily consumable protocol.

Push updates are produced by `vzlogger`. Updates consist of "raw" meter readings, that is the values that are produced by the source device. For example, an S0 meter will create "impulses"
They are sent to configured clients even if aggregation is configured. The protocol is JSON over HTTP.


````
            +-------------------------+--------------+
            |         vzlogger        |     httpd    |
            +-------------------------+--------------+
                        |                         ^
         push           |                         |  poll
    raw meter           | HTTP               HTTP |  converted
     readings           |                         |  readings
                        v                         |
            +-------------------------+      +---------------+
            |       push-server       |      | custom client |
            +-------------------------+      +---------------+
                ^              ^
    converted   |              |
        meter   | WAMP         | Web Sockets
     readings   |              |
                v              v
            +---------+    +----------+
            | Browser |    | node-red |
            +---------+    +----------+

````

Since the `push-server` provides conversion of raw meter readings into the usual "current" normalised values that the frontend is able to visualize, it needs to have access to the channel definition in order to chose the right interpretation logic.

As a consequence `push-server` must be considered part of the middleware:

  - it needs to share its installation with the middleware since components are reused
  - it is only able to provide updates for channels that are configured in the current middleware database

The `push-server` can however be on a separate machine from the actual vzlogger(s) or potentially any other clients that may generate raw meter readings.


## Data Formats

Push messages use a similar JSON data format to the regular middleware API less any parameters that are only required for more than one reading.

Sample JSON message:

````json
{
	"version": "0.3",
	"data": [
		{
			"uuid": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
			"tuples": [
				[
					1462369661000,    // timestamp in ms
					189.47368421053,  // converted value
					1                 // number of processed values (always 1)
				]
			]
		}
	]
}
````


## Configuration

### Basic Configuration

The `push-server` uses two ports for communication:

  - `$config['push']['server']` is the backend side port where `push-server` receives the raw readings from `vzlogger`. Default is 5555. This port is bound on `0.0.0.0` but can otherwise remain private as long as `vzlogger` can connect.
  - `$config['push']['broadcast']` is the server side "broadcast" port that clients can connect to using WAMP or plain web sockets. Default is 8082. This port needs to be accessible for any client that wants to receive push updates.


### Advanced Configuration

In order to limit the number of externally accessible ports, the web server can be configured to perform forwarding of client access to the HTTP port to the broadcast port based on certain criteria.

In the example of Apache below the web server is configured to forward requests to the `/ws` location to the broadcast port:

	LoadModule proxy_module modules/mod_proxy.so
	LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so

	<Location "/ws">
	    ProxyPass ws://localhost:8082
	</Location>

The `cboden/ratchet` web socket library that this implementation is based on does currently not support SSL (https://github.com/ratchetphp/Ratchet/issues/100). The forwarding technique demonstrated above can therefore also be utilized to forward incoming secure web socket  requests (`wss://`) to the internal unencrypted broadcast port (`ws://`).

If vhosts are used the `ProxyPass` directive needs be added to both http and https section.


### Client Configuration

The single port approach described above is the preferred setup for `push-server` deployments.

Volkszaehler frontend will automatically try to connect to `ws/wss:<middleware uri>/ws` for webservers that are configured accordingly. As fallback the `middleware.live` parameter in `options.js` is still available if a dedicated, separate port is required. In this case the `/ws` postfox is not appended to the URI.


### Routing

In order to support both WAMP and plain web sockets over the same external port, `push-server` has an integrated routing capability. For backwards compatibility with existing frontends the following routes are configured by default as part of `volkszaehler.conf.php`:

  - `$config['push']['routes']['wamp']`: array of routes for WAMP. Default is both `/` and `/ws` for frontend compatibility.
  - `$config['push']['routes']['websocket']`: array of routes for plain web sockets. Default is empty to prevent misuse. To enable chose e.g. `/socket` and make sure to read the chapter on security below.

To disable a protocol it is sufficient to declare an empty array of routes.


## Security


### WAMP

When exposing the `push-server` to internet (hostile) clients via WAMP clients gain the same level of read-only access as when using the frontend. That means that a client is only able to read data if he is in possession of a valid channel UUID that the client must subscribe to.


### Plain Web Sockets

Plain web sockets will currently forward **all** push notifications to connected clients. That implies that a client- once connected- obtains read access to all data updates and is also able to collect valid UUIDs for all updated channels, including UUIDs for non-public channels.

**NOTE** make sure you read and understand the above. If your volkszaehler installation allows frontend users to update or delete channels (or even to add/remove data) exposing plain web sockets will give away even private channel's UUIDs which can be misused.


## Installation

To install `push-server` as a service create the service using `sudo nano /etc/systemd/system/push-server.service` and paste the following template:

    [Unit]
    Description=push-server
    After=syslog.target network.target
    Requires=

    [Service]
    ExecStart=/usr/bin/php /var/www/volkszaehler.org/misc/tools/push-server.php
    ExecReload=/bin/kill -HUP $MAINPID
    StandardOutput=null
    Restart=always

    [Install]
    WantedBy=multi-user.target
