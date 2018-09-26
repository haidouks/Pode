# Pode

[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/Badgerati/Pode/master/LICENSE.txt)
[![Documentation](https://img.shields.io/badge/docs-latest-blue.svg)](https://badgerati.github.io/Pode)
[![Build](https://ci.appveyor.com/api/projects/status/nvl1xmh31crp10ea/branch/develop?svg=true)](https://ci.appveyor.com/project/Badgerati/pode/branch/develop)
[![Gitter](https://badges.gitter.im/Badgerati/Pode.svg)](https://gitter.im/Badgerati/Pode?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

[![Chocolatey](https://img.shields.io/chocolatey/v/pode.svg?colorB=a1301c)](https://chocolatey.org/packages/pode)
[![Chocolatey](https://img.shields.io/chocolatey/dt/pode.svg?label=downloads&colorB=a1301c)](https://chocolatey.org/packages/pode)

[![PowerShell](https://img.shields.io/powershellgallery/v/pode.svg?label=powershell&colorB=085298)](https://www.powershellgallery.com/packages/Pode)
[![PowerShell](https://img.shields.io/powershellgallery/dt/pode.svg?label=downloads&colorB=085298)](https://www.powershellgallery.com/packages/Pode)

[![Docker](https://img.shields.io/docker/stars/badgerati/pode.svg)](https://hub.docker.com/r/badgerati/pode/)
[![Docker](https://img.shields.io/docker/pulls/badgerati/pode.svg)](https://hub.docker.com/r/badgerati/pode/)

Pode is a Cross-Platform PowerShell framework that allows you to host [REST APIs](#rest-api), [Web Pages](#web-pages) and [SMTP/TCP](#smtp-server) servers. It also allows you to render dynamic files using [Pode](#pode-files) files, which is effectively embedded PowerShell, or other [Third-Party](#third-party-view-engines) template engines.

## Documentation

All documentation and tutorials for Pode can be [found here](https://badgerati.github.io/Pode). This documentation will be for the latest release, to see the docs for other releases, branches or tags, you can clone the repo and use `mkdocs`.

To build the documentation locally, the following should work (from the root of the repo):

```powershell
choco install mkdocs -y
pip install mkdocs-material
mkdocs serve
```

Then navigate to `http://127.0.0.1:8000` in your browser.

## Contents

* [Install](#install)
* [Documentation](#documentation)
    * [Setup](#setup)
    * [Docker](#docker)
    * [Basics](#basics)
        * [Specific IP](#specific-ip-address)
    * [REST API](#rest-api)
    * [Web Pages](#web-pages)
    * [SMTP Server](#smtp-server)
    * [Misc](#misc)
        * [External Scripts](#external-scripts)
        * [Certificates](#certificates)
    * [Helpers](#helpers)
        * [Attach File](#attach-file)
        * [Status Code](#status-code)
        * [Redirect](#redirect)
* [FAQ](#faq)

## Features

* Can run on *nix environments using PowerShell Core
* Host REST APIs and Web Pages
* Host TCP and SMTP server - great for tests and mocking
* Multiple threads can be used to response to incoming requests
* Use the full power of PowerShell, want a REST API for NUnit? Go for it!
* Ability to write dynamic files in PowerShell using Pode, or other third-party template engines
* Can use yarn package manager to install bootstrap, or other frontend libraries
* Setup async timers to be used as one off tasks, or for housekeeping services
* Ability to schedule async tasks using cron expressions
* Supports logging to CLI, Files, and custom loggers to other services like LogStash, etc.
* Cross-state runspace variable access for timers, routes and loggers
* Optional file monitoring to trigger internal server restart on file changes
* Ability to allow/deny requests from certain IP addresses and subnets
* Basic rate limiting for IP addresses and subnets
* Support for generating/binding self-signed certificates, and binding signed certificates
* Support for middleware on web servers
* Session middleware support on web requests
* Can use authentication on requests, which can either be sessionless or session persistant

## Install

You can install Pode from either Chocolatey, the PowerShell Gallery, or Docker:

```powershell
# chocolatey
choco install pode

# powershell gallery
Install-Module -Name Pode

# docker
docker pull badgerati/pode
```

## Documentation

This documentation will cover the basics on how to use Pode to create a simple REST API, Web Page, and SMTP Server. Further examples can be found in the examples folder.

### Setup

Pode has a couple of useful commands that you can use on the CLI, to help you initialise, start, test, build, or install any packages for your repo. These commands all utilise the `package.json` structure - as seen in Node and Yarn.

> At the moment, Pode only uses the `start`, `test`, `build` and `install` properties of the `scripts` section in your `package.json`. You can still have others, like `dependencies` for Yarn

```powershell
# similar to yarn and node, init will help you create a new package.json
pode init

# if you have a "start" script this will run that property; otherwise will run the script in "main"
pode start

# if you have a "test" script this will run that property
pode test

# if you have a "install" script this will run that property
pode install

# if you have a "build" script this will run that property
pode build
```

> By default, Pode will pre-populate `test`, `build` and `install` using `yarn`, `psake` and `pester` respectively

Following is an example `package.json`

```json
{
    "name":  "example",
    "description":  "",
    "version":  "1.0.0",
    "main":  "./file.ps1",
    "scripts":  {
        "start":  "./file.ps1",
        "test":  "invoke-pester ./tests/*.ps1",
        "install":  "yarn install --force --ignore-scripts --modules-folder pode_modules",
        "build": "psake"
    },
    "author":  "",
    "license":  "MIT"
}
```

### Docker

This is an example of using Docker to host your Pode scripts, using the `examples/web-pages.ps1` example from the examples directory. Below is an example `Dockerfile` to pull down the base container image, copy over the example files and then run the website (assuming this is run from the examples directory):

```dockerfile
# File: Dockerfile
FROM badgerati/pode
COPY . /usr/src/app/
EXPOSE 8085
CMD [ "pwsh", "-c", "cd /usr/src/app; ./web-pages-docker.ps1" ]
```

To build and run this, use the following commands:

```bash
docker build -t pode/example .
docker run -p 8085:8085 -d pode/example
```

Now try navigating to `localhost:8085` (or calling `curl localhost:8085`) and you should be greeted with a "Hello, world!" page.

### Basics

Pode, at its heart, is a PowerShell module. In order to use Pode, you'll need to start off your script by importing it:

```powershell
Import-Module Pode
```

After that, all of your main server logic must be wrapped in a `Server` block. This lets you specify port numbers, server type, and any key logic: (you can only have one `Server` declared in your script)

```powershell
# server.ps1
Server {
    # attach to port 8080
    listen *:8080 http

    # logic for routes, timers, schedules, etc
}
```

The above `Server` script will start a basic HTTP listener on port 8080. To start the above server you can either:

* Directly run the `./server.ps1` script, or
* If you've created a `package.json` file, ensure the `./server.ps1` script is set as your `main` or `scripts/start`, then just run `pode start`

Once Pode has started, you can exit out at any time using `Ctrl+C`. For some environments you probably don't want to allow exiting, so you can disable the `Ctrl+C` by setting the `-DisableTermination` switch on the `Server`:

```powershell
# server.ps1
Server {
    # logic
} -DisableTermination
```

> By default `Ctrl+C` is disabled in Docker containers due to the way input is treated. Supplying `-t` when running the container will allow exiting

#### Specific IP Address

You can use `listen` within your `Server` block to specify a specific IP, Port and Protocol:

```powershell
Server {
    # listen on everything for http
    listen *:8080 http

    # listen on localhost for smtp
    listen 127.0.0.1:25 smtp

    # listen on ip for https (and create a self-signed cert)
    listen 10.10.1.4:8443 https -cert self
}
```

### REST API

When creating an API in Pode, you specify logic for certain routes for specific HTTP methods. Methods supported are: DELETE, GET, HEAD, MERGE, OPTIONS, PATCH, POST, PUT, and TRACE.

> There is a special `*` method you can use, which means a route applies to every HTTP method

The method to create new routes is `route`, this will take your HTTP method, route, and logic. For example, let's say you want a basic GET `ping` endpoint to just return `pong`:

```powershell
Server {
    listen *:8080 http

    route 'get' '/api/ping' {
        param($session)
        json @{ 'value' = 'pong'; }
    }
}
```

The scriptblock requires a `param` section for just one argument: `$session`. This argument will contain the `Request` and `Response` objects; `Data` (from POST), and the `Query` (from the query string of the URL), as well as any `Parameters` from the route itself (eg: `/:accountId`).

The last line is to write the JSON response. Anyone hitting `http://localhost:8080/api/ping` will be greeted back with `{ "value": "pong" }`.

If you wanted a POST endpoint that created a user, and a GET endpoint to get details of a user (returning a 404 if the user isn't found), then it would roughly look as follows:

```powershell
Server {
    listen *:8080 http

    route 'post' '/api/users' {
        param($session)

        # create the user
        $userId = New-DummyUser $session.Data.Email $session.Data.Name $session.Data.Password

        # return with userId
        json @{ 'userId' = $userId; }
    }

    route 'get' '/api/users/:userId'{
        param($session)

        # get the user
        $user = Get-DummyUser -UserId $session.Parameters['userId']

        # return the user
        if ($user -eq $null) {
            status 404
        }
        else {
            json @{ 'user' = $user; }
        }
    }
}
```

> More can be seen in the examples under `rest-api.ps1`, and `nunit-rest-api.ps1`

### Web Pages

It's actually possible for Pode to serve up webpages - css, fonts, and javascript included. They pretty much work exactly like the above REST APIs, except Pode has inbuilt logic to handle css/javascript and other files.

Pode also has its own format for writing dynamic HTML pages. There are examples in the example directory, but in general they allow you to dynamically generate HTML, CSS or any file type using embedded PowerShell.

All static and dynamic HTML content *must* be placed within a `/views/` directory, which is in the same location as your Pode script. In here you can place your view files, so when you call the `view` function in Pode, it will automatically look in the `/views/` directory. For example, if you call `view 'simple'` then Pode will look for `/views/simple.html`. Likewise for `/views/main/simple.html` if you pass `'main/simple'` instead.

> Pode uses a View Engine to either render HTML, Pode, or other types. Default is HTML, and you can change it to Pode by calling `engine pode` at the top of your Server scriptblock

Any other file types, from css to javascript, fonts and images, must all be placed within a `/public/` directory - again, in the same location as your Pode script. Here, when Pode sees a request for a path with a file extension, it will automatically look for that path in the `/public/` directory. For example, if you reference `<link rel="stylesheet" type="text/css" href="styles/simple.css">` in your HTML file, then Pode will look for `/public/styles/simple.css`.

A quick example of a single page site on port 8085:

```powershell
Server {
    listen *:8085 http

    # default view engine is already HTML, so following can left out
    engine html

    route 'get' '/' {
        param($session)
        view 'simple'
    }
}
```

> This can be seen in the examples under `web-pages.ps1`

### SMTP Server

Pode can also run as an SMTP server - useful for mocking tests. There are two options, you can either use Pode's inbuilt simple SMTP logic, or write your own using Pode as a TCP server instead.

If you're using Pode's unbuilt SMTP logic, then you need to state so when creating the server: `Server -Smtp`. This will automatically create a TCP listener on port 25 (unless you pass a different port number).

Unlike with HTTP Routes, TCP and SMTP can only have one handler. To create a handler for the inbuilt logic:

```powershell
Server -Smtp {
    handler 'smtp' {
        param($session)
        Write-Host $session.From
        Write-Host $session.To
        Write-Host $session.Data
    }
}
```

The SMTP Handler will be passed a session already populated with the `From` address, the single or multiple `To` addresses, and the raw `Data` of the mail body.

> This can be seen in the examples under `mail-server.ps1`

If you want to create you own SMTP server, then you'll need to set Pode up as a TCP listener and manually read the SMTP stream yourself:

```powershell
Server {
    listen *:25 tcp

    handler 'tcp' {
        param($session)
        $client = $session.Client
        # your stream writing/reading here
    }
}
```

To help with writing and reading from the client stream, Pode has a helper function with two actions for `read` and `write`:

* `tcp write $msg`
* `$msg = (tcp read)`

### Misc

#### External Scripts

Because Pode runs most things in isolated runspaces, importing and using external scripts/modules to Pode can be quite bothersome. To overcome this, Pode has an inbuilt `script` call that will allow you to declare modules that need to be imported into each runspace.

The `script` takes a path to a module (`.psm1` file), can be literal or relative, and adds it to the session state for each runspace pool.

```powershell
Server {
    script './path/to/module.psm1'
}
```

> This will now allow the functions defined in the `module.psm1` file to be accessible to timers, routes, scheduled, etc.

#### Certificates

> Binding existing, and generating self-signed certificates is only supported on Windows

Pode has the ability to generate and bind self-signed certificates (for dev/testing), as well as the ability to bind existing - already installed - certificates for HTTPS. If Pode detects that the IP:Port binding already has a certificate bound, then Pode will not create a self-signed cert, or bind a new certificate - you'll have to clean-up the binding first: `netsh http delete sslcert 0.0.0.0:8443`.

For example, if you are developing/testing a site on HTTPS then Pode can generate and bind quick self-signed certificates. To do this you can pass the value `"self"` to the `-cert` parameter of `listen`:

```powershell
Server {
    listen *:8443 https -cert self
}
```

To bind a signed certificate, the certificate *must* be installed to `Cert:/LocalMachine/My`; then you can pass the certificate name/domain to `-cert`. An example for `*.example.com`:

```powershell
Server {
    listen *:8443 https -cert '*.example.com'
}
```

### Helpers

#### Attach File

`Attach` is a helper function to aid attaching files to a response, so that they can be downloaded on the client end. Files to attach must be placed within the `public/` directory, much like the content files for JavaScript and CSS.

An example of attaching a file to a response in a route is as follows, and here it will start a download of the file at `public/downloads/installer.exe`:

```powershell
Server {
    listen *:8080 http

    route get '/app/install' {
        param($session)
        attach 'downloads/installer.exe'
    }
}
```

#### Status Code

`Status` is a helper function to aid setting the status code and description on the response. When called you must specify a status code, and the description is optional.

```powershell
Server {
    listen *:8080 http

    # returns a 404 code
    route get '/not-here' {
        status 404
    }

    # returns a 500 code, with description
    route get '/eek' {
        status 500 'oh no! something went wrong!'
    }
}
```

#### Redirect

`Redirect` is a helper function to aid URL redirection from the server. You can either redirect via a 301 or 302 code - the default is a 302 redirect.

```powershell
Server {
    listen *:8080 http

    # redirects to google
    route get '/redirect' {
        redirect -url 'https://google.com'
    }

    # moves to google
    route get '/moved' {
        redirect -moved -url 'https://google.com'
    }

    # redirect to different port - same host, path and query
    route get '/redirect-port' {
        redirect -port 8086
    }

    # redirect to same host, etc; but this time to https
    route get '/redirect-https' {
        redirect -protocol https
    }

    # redirect every method and route to https
    route * * {
        redirect -protocol https
    }
}
```

Supplying `-url` will redirect literally to that URL, or you can supply a relative path to the current host. `-port` and `-protocol` can be used separately or together, but not with `-url`. Using `-port`/`-protocol` will use the URI object in the current Request to generate the redirect URL.

## FAQ

* Running `pode start` throws an `ImportPSModule` error.
  > This error occurs when you are using the source code for Pode, and also have the Pode module installed. To resolve you can do one of the following:
  > * Uninstall the Pode module from PowerShell, and re-`Import-Module` the source code version
  > * Manually call the `start` script
  > * Remove calls to `Remove-Module -Name Pode` within your scripts
