---
layout: docs
title: Graceful Start/Shutdown
description: How signals are handled in PM2
permalink: /docs/usage/signals-clean-restart/
---

## Graceful Stop

To allow graceful restart/reload/stop processes, make sure you intercept the **SIGINT** signal and clear everything needed (like database connections, processing jobs...) before letting your application exit.

```javascript
process.on('SIGINT', function() {
   db.stop(function(err) {
     process.exit(err ? 1 : 0)
   })
})
```

Now `pm2 reload` will become a gracefulReload.

### Configure the kill timeout

Via CLI, this will lengthen the timeout to 3000ms:

```bash
pm2 start app.js --kill-timeout 3000
```

Via [application declaration](http://pm2.keymetrics.io/docs/usage/application-declaration/) use the `kill_timeout` attribute:

```javascript
module.exports = {
  apps : [{
    name: 'app',
    script: './app.js',
    kill_timeout : 3000
  }]
}
```

## Graceful start

Sometimes you might need to wait for your application to have established connections with your DBs/caches/workers/whatever. PM2 needs to wait before considering your application as `online`. To do this, you need to provide `--wait-ready` to the CLI or provide `wait_ready: true` in a process file. This will make PM2 listen for that event. In your application you will need to add `process.send('ready');` when you want your application to be considered as ready.

```javascript
var http = require('http')

var app = http.createServer(function(req, res) {
  res.writeHead(200)
  res.end('hey')
})

var listener = app.listen(0, function() {
  console.log('Listening on port ' + listener.address().port)
  // Here we send the ready signal to PM2
  process.send('ready')
})
```

Then start the application:

```bash
pm2 start app.js --wait-ready
```

### Configure the ready timeout

By default, PM2 wait 3000ms for the `ready` signal.

Via CLI, this will lengthen the timeout to 10000ms:

```bash
pm2 start app.js --wait-ready --listen-timeout 10000
```

Via [application declaration](/docs/usage/application-declaration/) use the `listen_timeout` and `wait_ready` attribute:

```javascript
module.exports = {
  apps : [{
    name: 'app',
    script: './app.js',
    wait_ready: true,
    listen_timeout: 10000
  }]
}
```

### Graceful start using http.Server.listen

There is still the default system that hooks into `http.Server.listen` method. When your http server accepts a connection, it will automatically state your application as ready. You can increase the PM2 waiting time the listen using the same variable as `--wait-ready` graceful start : `listen_timeout` entry in process file or `--listen-timeout=XXXX` via CLI.

## Explanation: Signals flow

When a process is stopped/restarted by PM2, some system signals are sent to your process in a given order.

First a **SIGINT** a signal is sent to your processes, signal you can catch to know that your process is going to be stopped. If your application does not exit by itself before 1.6s *([customizable](http://pm2.keymetrics.io/docs/usage/signals-clean-restart/#customize-exit-delay))* it will receive a **SIGKILL** signal to force the process exit.

The signal **SIGINT** can be replaced on any other signal (e.g. **SIGTERM**) by setting environment variable **PM2_KILL_SIGNAL**.

## Windows graceful stop

When signals are not available your process gets killed. In that case you have to use `--shutdown-with-message` via CLI or `shutdown_with_message` in Ecosystem File and listen for `shutdown` events.

Via CLI:

```bash
pm2 start app.js --shutdown-with-message
```

Via [application declaration](/docs/usage/application-declaration/) use the `listen_timeout` and `wait_ready` attribute:

```javascript
module.exports = {
  apps : [{
    name: 'app',
    script: './app.js',
    shutdown_with_message: true
  }]
}
```

Listen for `shutdown` events

```javascript
process.on('message', function(msg) {
  if (msg == 'shutdown') {
    console.log('Closing all connections...')
    setTimeout(function() {
      console.log('Finished closing connections')
      process.exit(0)
    }, 1500)
  }
})
```
