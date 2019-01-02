---
title: "Logging with pino from docker"
date: 2019-01-02T10:05:00+01:00
categories:
tags:
- docker
- node
- pino
- papertrail
keywords:
- tech
# thumbnailImage: http://getpino.io/pino-tree.png
---

[Pino](http://getpino.io/#/) is a logging framework for node that promises *"Very low overhead"*. This article explains how to use it inside a docker container.

<!--more-->

## Pino

{{<alert info>}}
Sample code for this article can be found at https://github.com/mattiash/docker-pino-sample.
{{</alert>}}

The pino framework works by outputting all log-messages from your code
in a structured format on stdout.
You should then pipe the output from your program to a *transport*
that can format the logs and possibly also send it to a log storage.
At first, it might seem like a bad idea to serialize your logs to json
only to then parse it again,
but by doing this you actually off-load the job of transporting the logs to their final destination from your main-process onto another process.
Since node is by design (almost) single-threaded
this means that even though the total amount of work performed by the processor is higher with pino
(since it serializes and then deserializes all log-messages),
your main-process will have less work to do and will thus be able to do more actual work.

To use pino in your program, you do the following:

```javascript
const logger = require('pino')()

logger.info('hello world')
```

which produces the output

```
{"level":30,"time":1546421063300,"msg":"Listening","pid":9,"hostname":"15fc1a6eb602","v":1}
```

You then have to pipe the output from your program into a *transport*.
The simplest transport is `pino-pretty`:

```
$ node build/index.js | ./node_modules/.bin/pino-pretty -t
[2019-01-02 09:24:23.300 +0000] INFO  (9 on 15fc1a6eb602): Listening
```

The string `15fc1a6eb602` is supposed to be the hostname of the machine running the code.
If you run the code in docker it will be equal to the container id,
unless you specify a hostname with a `--hostname` parameter to `docker run`.

Piping the output of your program is an easy thing to do from the command-line,
but achieving the same thing in docker is more complicated.
In a previous article, I showed you how to [run node in a docker container](/2019/01/running-node-in-docker/).
The `entrypoint.sh` script contained the following:

```sh
#!/usr/bin/env sh

exec node build/index.js
```

The `exec` command is very important, since it makes sure that the TERM-signal is sent to the node-process instead of getting stuck in sh.
Unfortunately, just adding a pipe to the end of the exec-line will not work.
But we can solve the problem with a bit of bash-trickery with a new entrypoint.sh:

```bash
#!/usr/bin/env bash

pid=0

# SIGTERM-handler
term_handler() {
  if [ $pid -ne 0 ]; then
    kill -SIGTERM "$pid"
    wait "$pid"
  fi
  exit 143; # 128 + 15 -- SIGTERM
}

# on SIGTERM, kill the last background process, which is `tail -f /dev/null`
# and execute term_handler
trap 'kill ${!}; term_handler' SIGTERM

# the redirection trick makes sure that $! is the pid
# of the "node build/index.js" process
node build/index.js > >(./node_modules/.bin/pino-pretty -t) &
pid="$!"

# wait forever
while true
do
  tail -f /dev/null &
  wait ${!}
done
```

The script installs a handler for SIGTERM that allows us to handle the TERM-signal inside the bash-script.
It then starts node in the background and pipes its output to pino-pretty.

## Logging to papertrail

[Papertrail](https://papertrailapp.com/) is an online log-management system.
It allows you to gather the logs from all your servers and programs
into a centralized database where you can query them and
see how they all work.
It even allows you to setup email alerts that tells you if something unexpected happens.
It has a free-tier that is great for trying it out and running small-scale projects.

pino has a transport that forwards logs to Papertrail. Install it with

```
npm install pino-papertrail
```

and replace `pino-pretty -t` with the following

```plain
pino-papertrail --host papertrailhost --port papertrailport --appname myapp --message-only
```

You will get the values for `papertrailhost` and `papertrailport` when you sign up for papertrail.
Also remember to start your docker container with a sane value for the --hostname parameter since papertrail displays the hostname prominently in your logs.

Now, you will get access to the logs from your service at https://papertrailapp.com.
