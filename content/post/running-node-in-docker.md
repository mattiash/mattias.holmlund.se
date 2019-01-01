---
title: "Running node in docker"
date: 2019-01-01T13:05:00+01:00
categories:
tags:
- docker
- node
keywords:
- tech
#thumbnailImage: //example.com/image.jpg
---

Running your code in docker can give you reproducibility
and guarantee that your code does not depend on your working environment.
Getting it to run just right with tests and a small image size is however fairly complicated. This article explains how to do it with sample code.

<!--more-->

A good docker workflow should in my opinion satisfy the following requirements:

- Code is built in docker.
- Tests are run in docker.
- The resulting docker image for running the code only contains the files actually necessary for running the code.
- The docker container forwards signals to the daemon process and allows it to exit cleanly.

Achieving this for node-based projects is surprisingly difficult.
This article tries to explain how I do this for the images that I build.

The code for this article is available on github as [mattiash/docker-typescript-sample](https://github.com/mattiash/docker-typescript-sample)-

## Running modern javascript

In its simplest form, a node-based project consists of a set of javascript files
that you just have to run.
In most real-world applications, there are however two complications:

- A build step with more dependencies than the production code.
The build step must be run to produce the javascript-files that we want to run.
- Tests that should be run as part of the build process to make sure that the code still works.

The build-step differs between different projects, but typical examples are typescript and babel.
I use typescript, so that is what I will use in the sample code.

## Docker multi-stage builds

Docker 17.05 introduced [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/).
They allow you to write instructions for building several different docker images in a single Dockerfile,
and you can copy stuff between the different images.

## The base image

The first step in the build-process is to create a *base* docker image.
This image forms the basis for all other images that we use in our project.

```Dockerfile
FROM node:10 as base
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install --production
```

We derive the base-image from the official node image for nodejs 10.
To increase reproducibility, you might want to specify a stricter tag for node
(e.g. node:10.15.0).
We then copy package.json and package-lock.json to the container
and install the non-dev dependencies,
since we want them in all other images for this project.

## The build image

The next step is to create a *build* docker image.
The build-image includes all the dependencies that we need to build our code and test it.
We will throw away the build-image after we are done with it,
so we don't need to bother with keeping the size of the image down.

```Dockerfile
FROM base as build
WORKDIR /usr/src/app
RUN npm install
COPY . ./
RUN npm run build
```

The build image starts with our base-image and then copies the source-code of the entire
project into the image.

By running the `npm install` before we copy stuff into the container,
we allow the docker build to use the cached copy of node_modules if package.json hasn't changed.

It is possible to selectively copy the stuff that you need,
but I usually just copy everything.
However, I don't want to copy node_modules or any code built outside the container,
since then the contents of the resulting image would depend on whatever I had in my node_modules.
Therefore, I also have a `.dockerignore` file with the following contents:

```plaintext
node_modules
build
output
```

The `COPY . ./` actually means "copy everything except the stuff in .dockerignore".
My `tsconfig.json` is set up to generate all js-files in `build/` which makes it easy to ignore them here
and include them later as you will see.

```json
{
    "compilerOptions": {
        "module": "commonjs",
        "target": "es2016",
        "strict": true,
        "moduleResolution": "node",
        "inlineSourceMap": true,
        "baseUrl": ".",
        "paths": {
            "*": ["declarations/*"]
        },
        "outDir": "./build"
    },
    "include": ["*.ts"],
    "exclude": ["node_modules"]
}
```

The build-image will contain all our dependencies and our code and tests built.
We can now run our tests inside the build-image with

```shell
docker build --target build -t myproject-build:latest .
docker run --name myproject-build:latest npm run test
```

The second command will show the output from the tests in the shell.
I usually want the result of each individual test in an output file.
These files will be generated inside the docker container and we need to get them out somehow.
To do this, I have an npm script called `autotest` that runs the tests in a way that
generates a `.tap` file next to each test-file and.
I run the autotest-script from `autotest.sh`:

```bash
#!/bin/sh

rm -rf output
UUID=`uuidgen`
IMAGE=test
docker build --target build -t $IMAGE:$UUID .
docker run --name $UUID $IMAGE:$UUID npm run autotest
TEST_EXIT=$?
docker cp $UUID:/usr/src/app/build/test output
docker rm $UUID
docker image rm $IMAGE:$UUID
exit $TEST_EXIT
```

The script generates a random identifier (UUID) that is used both as the tag for the image
as well as for the name of the container that I run the tests in.
The script then builds the container with the `build` target
and runs the `autotest` npm-script inside the container.
All the generated tap-files are now inside the build/test/ directory
along with the built test-files.
Ideally, we only need to copy the tap-files from the container,
but `docker cp` does not support wildcards, so we just copy the entire directory
into the output/ folder in our local directory.
This allows us to process the tap-files in our ci-pipeline.
Finally, we make sure that the autotest.sh script exits
with the same exit code as the tests that we ran.

## Build production image

The final step is to build the production image.
We do this by first creating an image that contains everything from the build-image
except the tests (we don't want them in the production image):

```Dockerfile
FROM build as notest
WORKDIR /usr/src/app
RUN rm -rf build/test
```

and then we create the actual production image from the `base` image
and copy the stuff that we need from the `notest` image and our local checkout of the code:

```Dockerfile
FROM base as production
WORKDIR /usr/src/app
COPY entrypoint.sh ./
COPY --from=notest /usr/src/app/build ./build/
CMD ["./entrypoint.sh"]
```

The result is a production image that only contains the node_modules that we use in production,
the result of the build-step (minus test-scripts)
and our entrypoint file for running the code.

## Running the code

The final requirement for the docker container was 

- The docker container forwards signals to the daemon process and allows it to exit cleanly.

This means that when you issue a `docker stop`command,
a signal shall be forwarded to the javascript code that allows it to
cleanup and close all resources properly before exiting.

The index.ts file in the sample repository looks like this:

```typescript
import { createServer } from 'http'

const server = createServer((_req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' })
    res.end('ok')
})

server.listen(3000, () => {
    console.log('Listening')
})

process.on('SIGTERM', () => {
    console.log('Shutting down')
    server.close(() => {
        console.log('Socket closed')
    })
})
```

The code is a very simple http-server. When the user issues a `docker stop` command,
the process receives a TERM-signal and shuts down.
After receiving the TERM-signal, the javascript code shall exit without any further action from docker.
If it does not exit, the docker daemon will kill it unconditionally after a timeout.

Now we need to make sure that the TERM-signal actually reaches the node-process. 

The `docker kill` command sends a TERM signal to the *main* process (PID 1) in the container.
The simplest way to start node in the container is to end the Dockerfile
with a

```Dockerfile
CMD node build/index.js
```

Under the hood, this results in an sh-process as the main process that in turn starts the node process.
Unfortunately, the sh-process does not forward signals,
which means that the TERM-signal will not be sent to node.

To make it work, you have to use the execform of CMD

```Dockerfile
CMD ["node", "build/index.js"]
```

Sometimes you need to pass more parameters to node,
set environment variables or something else before you start node.
To make this easier, I always have an `entrypoint.sh` that starts node:

```Dockerfile
CMD ["./entrypoint.sh"]
```

and an entrypoint.sh:

```sh
#!/usr/bin/env sh

exec node build/index.js
```

Note that the file ends with an exec-command that replaces the sh-process
with the node-process.
This is needed to allow docker to send the TERM-signal to node.

## Using the container

To actually build and use the container, there is a `build_production.sh` script that builds and tags the container:

```sh
#!/bin/sh

docker build -t docker-typescript-sample:latest .
```

Since the production image is the last part of the Dockerfile,
it will be built if we don't specify a `--target` parameter.

Finally, the `run_container.sh` script:

```sh
#!/bin/sh

docker run --init -d -p 3000 --rm --name docker-typescript-sample docker-typescript-sample:latest
```

The `--init` parameter makes sure that docker starts an init-process inside the container
that handles zombie-processes.
This is only needed if the node-process starts sub-processes (e.g. with spawn),
but it is good practice to always use it since it can lead to strange problems
if you forget it.

The `--rm` argument tells docker to delete the container when it is stopped,
which is fine since you should not be storing any data inside the container anyway.

It is now time to test that the container actually behaves as it should.
Run build_production.sh followed by run_container.sh.
Test that the daemon responds by opening http://localhost:3000 in a browser.
Start 

```
docker logs --follow docker-typescript-sample
```

in one shell. Run 

```
docker stop docker-typescript-sample
```

in another shell.
The docker stop command should not take more than a second and you should see

```
Listening
Shutting down
Socket closed
```

from docker logs before it exits.
This tells us that the daemon was shut down correctly.

## Continuous Integration

Now you should take all these parts and tell your ci-system to run the following steps:

- Run ./autotest.sh
- Parse the test-output in output/ and check for errors
- Run ./build_production.sh
- Run docker stop docker-typescript-sample
- Run ./run_container.sh

That exercise is up to you to perform.
