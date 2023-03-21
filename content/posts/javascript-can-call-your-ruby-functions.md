---
title: "JavaScript can call your Ruby functions directly"
date: "2020-02-13"
draft: false
description: A weird deep dive into how JavaScript and Ruby can interact.
summary: Join me on my wonky journey learning about invoking Ruby functions in a JavaScript runtime.
categories:
- Code and Tech
tags:
- JavaScript
- Ruby
- Weird Hacky Things
---

Wait, really? Yes, really.

I recently needed to solve an odd problem that required this kind of weird solution. Unfortunately, the solution fell short of the exact need for my client; however, the results of my findings are definitely worth sharing — especially given that my PopularSearchEngine™ deep dive yielded very little to help me figure all of this out.

If you follow me on Twitter, you may have seen my cryptic tweet after spending hours trying to get this to work:

> Today… I wrote a #Rails script that creates a #CommonJS env with a #V8 runtime that attaches a #Ruby function, loads a #Webpack bundle, requires a custom #module, and calls a #javascript function with the results from the Ruby function directly in the JS. 🤷🏻‍♀️
>
> How’s your day?— Laura Mosher (@LauraTrev) January 24, 2020

Therefore, this blog post is a BehindTheTweet™ deep dive and a how-to guide so you can do this yourself. So, if you find this kind of bizarre Ruby and JavaScript interesting or you are trying to simulate the same kind of result, read on.

## The Problem
The framework the rest of this guide follows stems directly from the requirements that I was presented with by my client. In order for everyone to have the same frame of reference, this is a brief background on the codebase and the ask.

### The Codebase Setup
The backend built with Ruby on Rails and the frontend is an Angular app. The Frontend lives in its own folder inside the root of the Rails codebase. When deployed, The frontend is compiled with Webpack into static assets and loaded into Rails via <script> tags in the Rails application layout view file.

### The Ask
There exists a set of data that was in one format and the goal was create a script that migrated the data from that format into the new format and save it. The catch? The data was stored in the Rails database and the code to transform it and generate the format lived in the Angular code. The question of data flow was uncertain: The frontend didn’t know how to save the data and the backend had no idea how to do the transformation correctly.

They wanted a modular and reusable way to run data migration scripts that required Frontend code, and they wanted to keep it all in a rake task, if possible.

## Prerequisites
Everything can be done with Ruby + a Frontend app; that said, since the code I was working in was expected to work inside the context of Rails, this guide will also be done within Rails.

This how-to assumes the following:

1. You have Rails app Backend with at least on Model.
1. You have a modern Frontend app.
1. You use Webpack to compile your Frontend assets.

## Setup a JavaScript environment in your Rails environment
The first step is to get our Rails environment setup to have an internal JavaScript runtime environment.

### Install Gems
There are two gems that are needed to create a (mostly modern) V8 runtime and CommonJS environment. These will allow us to run JavaScript in our Ruby script:

```ruby
# In Gemfile
group :rake do
  gem 'commonjs-mini_racer_env', github: 'tribune/commonjs-mini_racer_env'
  gem 'mini_racer'
end
```

[MiniRacer](https://github.com/rubyjs/mini_racer) is a minimal modern embedded V8 runtime for Ruby. When coupled with [CommonJS for Mini Racer](https://github.com/tribune/commonjs-mini_racer_env/), we can evaluate reusable JavaScript modules in a non-browser environment directly in Rails.

### Creating and Running your Script
Rails has two primary ways to run snippets of code server side, in-context of Rails: A rake task or a Ruby script with Rails `runner`. I’ve found that I prefer to use the latter option for most things, unless I want to be able to reuse the code inside other Rake tasks, and for a migration of data, having a separate script felt appropriate. That said, either option will work. For the purposes of this guide, we will use the script + Rails runner pattern.

Lets create the shell of our script:

```ruby
# in db/scripts/my-script.rb
puts "Hello world!"
```
And then run it to make sure everything is wired correctly:

```sh
$ rails runner db/scripts/my-script.rb
=> Hello world!
```
### Initialize the JavaScript environment
Now that we’ve verified that everything runs as expected, lets setup our JavaScript environment with the two gems we installed:

```ruby
# in db/scripts/my-script.rb
require 'commonjs-mini_racer_env'

# Define our V8 Runtime context
context = MiniRacer::Context.new

# Create our JS Environment
js_env = CommonJS::MiniRacerEnv.new(
  context,
  path: Rails.root.join('app', 'javascripts')
)
```
### Verify your JS environment
The quickest way to see if you have a successful JS runtime is to evaluate a small bit of JavaScript code after your created JS Environment and run the script:

```ruby
puts js_env.runtime.eval('1+3')

# When you run the script, you should see:
=> 4
```
Or recreate the “Hello world!” example, except this time with JavaScript using a multiline heredoc:
```ruby
js = <<-JS
function hello() {
  return 'Hello World!';
}
hello();
JS

puts js_env.runtime.eval(js)

# When you run the script, you should see:
=> Hello World!
```
If you see the expected results from these examples, you’re good to go!

## Linking Ruby functions into your JS environment
Okay so, so far all we’ve established is that we can run JavaScript code in Rails. While neat, that’s not how I caught you with the title. I said you could call Ruby functions _from_ JavaScript. Lets do it: transform the Hello World! example to instead return Hello, \<name\>! where the fetched name comes from Ruby.

### Define the Ruby function
Lets create a function that fetches a random user from the database when called.

Note: In order to allow synchronous calls to Ruby from JavaScript, the Ruby functions needs to be defined as a proc function.

```ruby
fetch_random_user = -> {
  User.all.sample&.as_json
}
```
### Attach it to the JS Runtime
Now that we have our proc function, we can attach it to the JS runtime:
```rb
js_env.runtime.attach(
  "Ruby.fetch_random_user",
  fetch_random_user
)
```

This attaches our `fetch_random_user` ruby proc to our JS runtime at the definition Ruby.`fetch_random_user()`. Ruby is optional and can be named anything you would like (or omitted entirely). I prefer to prefix my Ruby functions in the JS environment with it, or something similar, so I know that it is defined in Ruby at usage, rather than somewhere in my JS.

### And execute on it
We can now utilize the attached Ruby function directly in our JavaScript and pass the results into a JavaScript function and output the results from Ruby.

```ruby
js = <<-JS
function hello(user) {
  return `Hello, ${user.name}!`;
}

var user = Ruby.fetch_random_user();
hello(user);
JS

puts js_env.runtime.eval(js)
#> Hello, Laura!
```

## Using your Frontend app assets
The previous portion showed how to run generic Ruby functions in a JavaScript runtime, with inline defined JavaScript functions defined in the Ruby script. This next section is going to show you how to you can use your Frontend assets and defined functions in that same runtime.

### An Example App
Lets say we have a small application that does the following:

1. When given a User, the program says hello to the user, like we did in the previous example, and transforms the user’s data into a special json structure for a new user Profile.
1. When a user is not provided, the program should exit with a generic ‘Hello world!’ and a message stating that it cannot generate a profile.
1. The format for the Profile json should look like the following:

  ```json
  {
    "profile": {
      "name": "Laura",
      "favorites": {
        "animal": "hedgehog",
        "color": "teal"
      },
      "hobbies": [
        "reading",
        "running",
        "climbing"
      ]
    }
  }
  ```
### Create Modules
Lets create two modules, one for saying hello:

```js
// in src/hello.js
export function hello(user) {
  return `Hello, ${user.name}!`;
}

export function helloWorld() {
  return 'Hello world!';
}
```
and one for holding the transform logic:

```js
// in src/transform.js
export function tranform(user) {
  let profile = {
    profile: {
      name: user.name,
      favorites: {
        animal: user.animal,
        color: user.color
      },
      hobbies: user.hobbies
    }
  };

  return JSON.stringify(profile);
}
```
### Create main application
We can import those modules directly into our primary application file:

```js
import { hello, helloWorld } from "./src/hello.js";
import { transform } from "./src/transform.js";
```
And then create a few functions to handle our main logic:
```js
export function main(user) {
  let greeting = sayHello(user);
  let profile = transformProfile(user);

  return [greeting, profile].join('\n');
}

function sayHello(user) {
  if (user === undefined) {
      return helloWorld();
  }
  return hello(user);
};

function transformProfile(user) {
  if (user === undefined) {
      return "Unable to generate Profile for this user";
  }

  return transform(user);
}
```
## Bundle Your Frontend
For us to use our new and lovely frontend application, we need to create a Webpack bundle that we can import into our runtime. Here is a barebones example of a custom webpack configuration file:
```js
// my-webpack.config.js

var path = require('path');

module.exports = {
  target: 'node',         // [1]
  entry: {
    'main': './main.js',
  },
  output: {
    filename: '[name].js',
    library: 'YourLibraryName',
    libraryTarget: 'umd', // [2]
    path: path.resolve(__dirname, '../public/assets')
  },
}
// [1] Targetting node is important so the bundle is
//     self-contained and can be included in our Ruby script!
// [2] umd stands for Universal Module Definition, and is
//     necessary to load into our CommonJS environment.
```


Once you have your configuration setup, you can create your target library with:
```sh
webpack --config my-webpack.config.js
```
This will create a main.js file available in the rails public/assets folder. It will include all of the imported bundles

### Load your bundle into the JS Environment
Using the JS environment you created in your Ruby script, you can now load your newly compiled bundle library into your runtime:

```rb
# Load your JS bundle
js_env.runtime.load(
  Rails.root.join('public', 'assets', 'main.js')
)
```
### Use your bundled modules
Now you can use your bundled modules using CommonJS && the Node require() function:
```js
// if you want to require the full Module / Library
var YourLibraryName = require('main');

// if you want to require just the `main` function:
var { main } = require('main');
```

## Putting it all together

```rb
# In db/scripts/my-script.rb
require 'commonjs-mini_racer_env'

context = MiniRacer::Context.new

js_env = CommonJS::MiniRacerEnv.new(
  context,
  path: Rails.root.join('app', 'javascripts')
)

js_env.runtime.load(
  Rails.root.join('public', 'assets', 'main.js')
)

fetch_random_user = -> {
  User.all.sample&.as_json
}

js_env.runtime.attach(
  "Ruby.fetch_random_user",
  fetch_random_user
)

# successful case
js = <<-JS
var YourLibraryName = require('main');
var user = Ruby.fetch_random_user();
YourLibraryName.main(user);
JS
puts js_env.runtime.eval(js)
#> Hello, Laura!
#> {"profile":{"name":"Laura","favorites":{"animal":"hedgehog","color":"teal"},"hobbies": ["reading","running","climbing"]}}

# unsuccessful case
js = <<-JS
var { main } = require('main');
main();
JS
puts js_env.runtime.eval(js)
#> Hello world!
#> Unable to generate Profile for this user
```

### Next Steps: Accepting args & saving data
At this point, we’ve shown how you can call Ruby functions from a JS runtime context and how you can load frontend module assets into a CommonJS environment. You can take it a step further have your Ruby functions accept arguments and save data.

An example of how that might look:
```rb
save_user_profile = ->(user_id, user_profile) {
  user = User.find(user_id)
  user.profile = user_profile
  user.save
}
```
## Notes && Caveats
Like all solutions, your mileage may vary. A few known caveats based on my testing:

- This method does not work if you need access to the window or dom. This ultimately was why this solution did not work for my client.
- Why not ExecJS or TheRubyRacer? I ruled out both relatively early on. You can use ExecJS for JS runtime without the need for bundled JS assets. TheRubyRacer’s V8 runtime is old and misses a lot of the niceties we’ve come to know and love and rely on (Like promises!). You can use it if you don’t need anything beyond V8 version 3. If you are able to use TRR, you can also use the default CommonJS gem (and not the one specifically ported for MiniRacer)!

## Good luck and have fun!
I hope you enjoyed this forary into Ruby inside JavaScript.