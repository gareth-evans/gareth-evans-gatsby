---
path: debugging-node-in-docker-with-visual-studio-code
date: 2016-06-23T12:52:05.482Z
title: Debugging Node running in Docker with Visual Studio Code
description: "I’ve just started looking into Docker so I am no means an expert.
  One of the first questions I had was how do I debug an app that is running in
  a docker container. The answer is Visual Studio Code and it’s actually quite
  easy. "
tags:
  - Docker
  - VisualStudioCode
redirects:
 - /2016/06/23/Debugging-Node-running-in-Docker-with-Visual-Studio-Code/
---
In this post I’ll walk you through a sample project that is available from [here](https://github.com/gareth-evans/node-debug-in-docker).

## The Application

The app that we are going to debug is a very simple (and pretty useless) HTTP server. The server will respond with “Hello World” to any request.

```javascript
var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end("Hello World");
}).listen(9615);
```

## Building Our Image

In the spirit of the application the docker file is also very simple, but for those of you who are not familiar with docker I’ll go through it a line at a time.

```dockerfile
FROM node:latest
RUN npm install -g nodemon@latest
WORKDIR /var/www
EXPOSE 9615
ENTRYPOINT ["nodemon", "-L", "--debug", "server.js"]
```

1. Indicates that we will use the latest node image as a base
2. Install nodemon globally into the image
3. Set the working directory in which the entry point is executed. This path won’t exist in the image but we’ll fix that when create the container.
4. Expose the port on which our server is listening.
5. The command and arguments that will be called when the container starts. We’re using nodemon so that if a change in the source files is detected then node will be restarted.

With the docker file in place we can build the image using the following command executed from the location where the Dockfile resides.

```powershell
docker build -t myapp .
```

Once the image has built successfully you can start the container using the following command.

```powershell
docker run -p 9615:9615 -p 5858:5858 -v $(pwd):/var/www -d myapp
```

The interesting points here are these:

\-p 9615:9615 – maps the http listening port of the container to the host port\
-p 5858:5858 – maps the node debug port to the host\
-v $(pwd):/var/www – maps the current working directory to the path /var/www in the container. Remember we set the working directory to /var/www in the Dockerfile\
myapp – The name of our image created previously\
We now turn our attention to setting up Visual Studio Code for debugging. Below is the launch configuration file which allows us to attach to the node process running in the Docker container.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach",
            "type": "node",
            "request": "attach",
            "port": 5858,
            "address": "192.168.99.100",
            "restart": false,
            "sourceMaps": true,
            "localRoot": "${workspaceRoot}",
            "remoteRoot": "/var/www",
            "outDir": null
        }
    ]
}
```

The interesting points in this file are these:

1. The port on which VS Code will attempt to connect to the running node instance
2. The IP address of the docker host (or IP address of the container on Linux)\
   13.The location of the source files inside the container

With all this setup you can set a breakpoint at line 4 in server.js, hit F5, navigate to [http://192.168.99.100:9615](http://192.168.99.100:9615/) (or whatever the IP is for your host or container) and you should hit your breakpoint.

If you have any problems with this let me know in the comments.