---
layout: post
title:  "Dynamic log level update at runtime"
description: Survey of support and manual techniques for dynamically changing log level of an application at runtime
img:
date: 2023-06-15  +1500
---

A small survey of existing techniques and support for dynamic (run-time) external modification of logging level of an application.
While most logging libraries do allow changing the log level from the code, consider a case where an application is already running and we want to change the log level. We want to do this at run-time itself since we may not not want (or be able) to change the code or re-compile. Moreoever, sometimes we want to debug a problem that has already happened, and we might lose that state if we re-start the application.

Popular languages already have some support for this, either a particular logging library or otherwise (for Java). For languages without support, we might need some manual workarounds or implementation to enable this. I cover both existing support and how to implement manually in the below sections.

## Existing support
### Python
Python's default logger has a [listen config](https://docs.python.org/3/library/logging.config.html#logging.config.listen) which starts a socket server on a particular port. Then, the logger configuration can be modified dynamically by passing the [Config dict struct](https://docs.python.org/3/library/logging.config.html#logging-config-dictschema) in JSON form.

It also has an `incremental` change option in the configuration to handle incremental changes in level. This means that we do not have to pass the entire config again, but just the new level along with the `incremental` flag to set the new level.

So, for debugging your own application, you can start the config listen socket, and dynamically change the log levels as per requirements.

### Golang
Uber has a logger (in Golang) called [zap](https://github.com/uber-go/zap). It is supposed to be fast and allows structured logging, but personally I feel that it seems quite complicated for something as simple as logging (especially due to the structured logging ability).

If we see the [documentation](https://pkg.go.dev/go.uber.org/zap#section-documentation), they offer multiple choices for type of logger (simple or a faster Sugared logger) and multiple configurations (production or dev config) and so on.

Zap introduces something called an AtomicLevel, which is a dynamically (runtime) changeable log level.
It also serves an HTTP endpoint, allowing to use a REST API for changing the levels.

We can see their implementation of the HTTP handler [here](https://github.com/uber-go/zap/blob/master/http_handler.go#L71) and the line above gives an example of how to use the HTTP API to update the log level.

Unlike the python option, this requires us to either be using the Zap logger or change our logger to Zap. Further, it will also require some code changes since the default logger is different from the `AtomicLevel` logger.

### Java - multiple techniques
[This blog](https://www.baeldung.com/spring-boot-changing-log-level-at-runtime) covers some different techniques for Java, I have summarized the main techniques below.

#### Springboot actuator logging endpoint
Springboot Actuators have "Endpoints" allowing one to configure and see different parts of the application. These endpoints can be enabled and exposed over a HTTP server. Logging is also an endpoint. With this functionality, one needs to simply expose the "logging" endpoint, which will allow changing the configuration through a HTTP API. [This blog](https://ryanharrison.co.uk/2021/01/30/dynamically-change-log-level-runtime-spring-boot.html) covers this in some detail.

#### Logback scan for changes
[Logback](https://logback.qos.ch/manual/configuration.html#autoScan), a logger for Java, has an option of automatically scanning for changes in the config file and reloading config in case of modifications. This requires setting a config option of `scan`.

### Rust
#### log4rs scan for changes
[`log4rs`](https://docs.rs/log4rs/latest/log4rs/) is a logger modelled after the Java Logback logger. So similar to the logback scan for changes, we specify the configuration through a `yaml` file. The config also has a `refresh_rate` parameter, which specifies how often the file must be checked for changes. So we can simply change the level in the file and it should be reflected the specified seconds later.

#### tracing
tokio's `tracing` is a Rust crate for event driven and structured diagnostics. (Aside - it makes sense that tokio cares about this, it is the primary asynchronous programming library in Rust). While it is mainly a tracing library and focusses on events, they have some options to also log the events. So, this is not technically a "logging" library, but a tracing library which also has logging functionality.

It has a [`reload` layer](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/reload/index.html) which gives a handler on per-layer loggers and allows to change the level, but does not have a HTTP API over it. I think this is more about the fact that it provides different layers whose configuration needs to be changed independantly.

So it does NOT have dynamic reload out-of-the-box.
[This code example](https://github.com/tokio-rs/tracing/blob/master/examples/examples/tower-load.rs) shows how they have implemented a HTTP handler on top of the reload functionality to get run-time configurable reload. 
## Other techniques
### Write your own server
One of the simplest ways to implement dynamic logging manually is the approach that most of the above techniques use. Basically, we need to start a server exposing an HTTP endpoint, which takes in requests and internally calls the `set_level()` equivalent function for your logger. This works since most loggers allow to change the log level through the code.

### Signal to read changed config file
In some cases, an HTTP server just for log level changing might be too much of an overhead. A lesser overhead solution is the Logback solution, where we want to put the config in a file and somehow signal that the file is changed and to reload the config.

So, like Logback we could start a thread that periodically scans the config file and updates the config.
Another approach would be to have a signal handler which reloads the file upon receiving a particular signal. More on that class of techniques below.

### Signal handling
A fundo, though slightly (read very) hacky method for doing this without files is using the `SIGUSR1` and `SIGUSR2` signals.

[One SO answer](https://codereview.stackexchange.com/questions/196175/dynamically-change-logging-level-of-a-python-process) uses these signals to directly set the log level as either DEBUG or INFO.

Another method used the signals to either decrease or increase the log level. I am sure I saw this somewhere on SO but am not able to find it. So I decided to implement this and try for myself.

The following is an implementation in python (building on the above SO answer), where the `SIGUSR1` signal will increase the log level and `SIGUSR2` signal will decrease the log level.

```python
#!/usr/bin/python
import logging
import time
import signal

logging.basicConfig(level=logging.ERROR)
logger = logging.getLogger(__name__)

logLevels = {
        0: logging.DEBUG,
        1: logging.INFO,
        2: logging.WARN,
        3: logging.ERROR
    }

currentLevel = 3

# SIGUSR1 => increment log level 
def handler1(signum, frame):
    global currentLevel
    if currentLevel != 3:
        currentLevel += 1
    logger.setLevel(logLevels[currentLevel])

# SIGUSR2 => decrement log level
def handler2(signum, frame):
    global currentLevel
    if currentLevel != 0:
        currentLevel -= 1
    logger.setLevel(logLevels[currentLevel])

signal.signal(signal.SIGUSR1, handler1)
signal.signal(signal.SIGUSR2, handler2)

while True:
    logger.info('info logging')
    logger.debug('debug logging')
    logger.error('error logging')
    logger.warn('warn logging')
    time.sleep(2)

```

Probably not the most efficient, I use a global variable to keep track of the current log level (somehow python logging library does not have a `getLevel()` function) and a map from the number to the log level.

We can use `kill -s USR2 <pid>` to see the log level go from `ERROR` to `WARN` to `INFO` and so on.

## Tangential notes
### Log performance benchmarking
Uber's zap has a [section on performance of logging libraries](https://github.com/uber-go/zap#performance) which has a table comparing different logging library performances. But it is interesting to note the overhead of logging over not logging. For instance, the `logrus` library that I use has a 1200% overhead on performance.

### Aside - Understanding Zap (WIP)
Somehow Zap feels quite complicated for something as simple as a logging library. Zap explains the reasons why loggers could have bad performance and what it does to solve it.

One aspect is using `encoding/json` to print JSON.

It also does sampling (for production config). There are many questions I have here - like is this sampling lossy or lossless. Are only pure repeat messages dropped? As per their description, they only drop repeat entries or duplicate messages.
For instance, in [sampler.go](https://github.com/uber-go/zap/blob/382e2511e51cda8afde24f9e6e741f934308edfa/zapcore/sampler.go#L152), they describe their sampler as follows: in every time tick, sample first N logs. If it exceeds, pick every Mth log entry. This seems to be actually lossy.
