---
title: "Getting Node.js Working on Heroku: Abridged"
date: '2018-09-15'
draft: false
description: A lesson in Heroku Ports with Node.js
summary: I recently ran into an issue with hosting a Node.js on Heroku. Turns out, Heroku needed a little extra configuration.
categories:
  - Code and Tech
tags:
  - JavaScript
  - Node
  - Heroku
---

Once in a while, I get to work on a small, one-off project that solves some need. Recently, I got to write one with one of my colleagues and we decided to host it on Heroku.

Unfortunately, the setup wasn’t quite as straightforward as we might have hoped.

Our app is built in Node.js with a web component in Express and a CLI component that can be run through yarn via the command line. We used Redis for datastore and config for environment configuration management. Heroku let us have the control we needed to run both as 2 modules in the same repository and application.

Here are the things we wish we knew before we pushed to production.

## PORTS
Tip: Make sure you are using the PORT Heroku provides you.

A basic node+express application usually follows this general setup to instantiate the server:

```js
const express = require("express")

const app = express()

app.start = async () => {
  const port = 8080
  app.set("port", port)

  const server = http.createServer(app)

  server.listen(port)
}
```

If you were to push this up to Heroku, it would not be able to bind to the specified port, 8080. This is because when Heroku spins up an application, it assigns a port for you and expects your application to respond at that port.

How do you get the port that Heroku assigns? Well, they place your assigned port into your apps environment variables. You can use it like so:

```js
app.start = async () => {
  const port = process.env.PORT
  ...
}
```

Okay, that’s all well and great, but I still need a port for development and testing but I don’t want to have to export the PORT environment every time I spin up locally.

```js
const port = process.env.PORT || 8080
```

## config package in Production

> Tip 1: If you need custom environment variables in your configuration, use the proper config file.

The config package is a neat tool for managing configuration of multiple environments. This app only had two: development and production, but even for this simple use case it was extremely helpful.

This app specifically needed this level of configuration because we integrated with Slack API and GoogleSheets API — both needing tokens and other sensitive information that is best left out of the codebase.

In development, the codebase has a committed config/development.json.example and have the developer set up with the proper tokens.

In production though, those values are set in environment variables. Per the documentation, custom environment variables have their own file config/custom-environment-variables.json organized in the same structure as your configuration.

> Tip 2: Ensure you are also specifying a config/production.json config file so the server knows there is a production environment, even if all of your configuration is in that special file.

The catch? Even if every single one of your configurations utilize a custom environment variable, you still need to have a config/production.json file for the sever to run.

## redis configuration
When you’re working with redis locally, the most common way of creating the connection is to do the following:

```js
const client = redis.createClient(redisConfig.host, redisConfig.port)
```

But when you provision Heroku Redis for your app, Heroku provides the environment variables REDIS_URL. Technically, it includes the host and port, as well as a user name and password, but the provided url includes everything you need.

So instead, this is the HerokuWay™ for creating the Redis client.

```js
const client = redis.createClient(redisConfig.redisUrl)
```
If you’re using config for your environment variable configuration, update your default configuration to look like this:

```json
{
  "redis": {
    "redisUrl": "redis://127.0.0.1:6379/1"
  }
}
```
And your custom environment variable configuration

```json
{
  "redis": {
    "redisUrl": "REDIS_URL"
  }
}
```
No need to add it to your production config since it inherits from default.

## Heroku Scheduler
> Tip: You can use a guard clause in your code to only run a Heroku Scheduler task on the day you wish it to run.

Heroku Scheduler only runs at 3 frequencies: Daily, Hourly, and every 10 minutes. We needed something that ran once weekly.

Since we were on a time crunch, and we didn’t need absolute precision, we decided rather than write our own clock process, we could write a guard clause in the code and then set up a daily task.

This meant we were running a daily task, but only allowing the full command to run if it was the proper day. This worked great.

## Heroku Scheduler: Take 2
But, it wasn’t guaranteed to run.

> Tip: If you need consistency, write a custom scheduler, or use a different add-on. Heroku Scheduler makes no guarantee on running.

The first few times, the task worked flawlessly. And… completely messed up the next. In reality, we had set up 4 separate daily tasks. We ran the two commands twice on the specified day: 1 to set up, 1 to execute 2 hours later, and then repeat again later in the day. The first 2 tasks finished. The 3rd one failed, or the second set up task. So when the 4th task started, and worked, it took the setup criteria from the FIRST task and used it — causing all sorts of confusion.

So, this is your ADDITIONAL reminder (since Heroku Scheduler does tell you this when you’re look at its page) that it is not guaranteed, and to write your own clock process if you need precision and control.

That’s all, folks!