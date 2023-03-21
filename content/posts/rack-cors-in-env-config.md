---
title: Rack CORS origins via ENV var and Regex
date: "2020-10-19"
draft: false
description: Configure CORS per environment in Rack apps with Rack CORS.
summary: Supporting multiple origins with variable URLs can be frustrating. Rack CORS and environment variables makes it easier.
categories:
- Code and Tech
tags:
- Ruby
- Rails
- Environment Configuration
- Rack
- CORS
---

Rack CORS is a Ruby [gem](https://github.com/cyu/rack-cors) that you can use to enable Cross Origin Resource Sharing (CORS) support in your application. CORS errors are common issues when you have an API-only Rails Application with a separate front end.

The basic configuration looks like:

```rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "*"

    resource "*",
      headers: :any,
      methods: :any
  end
end
```

In this example, we’ve set `origins` to `"*"` which will:

- allow ***any*** origin
- to hit ***any*** of the resources in our Rails application
- with ***any*** headers, and
- ***any*** methods.

Any. Anything. Or everything, depending on how you look at it.

If you search for CORS issues with Rails, many of the top hits suggest setting origins to “*” to get unblocked. This works; however, it leaves your API open to anything and anyone connecting.

## Restricting to Specific CORS Origins
Fortunately, the gem provides us with a way to specify specific origins, and multiple of them.

```rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "localhost:3000", "https://my.staging-app.com"

    resource "*",
      headers: :any,
      methods: :any
  end
end
```
And, you can use regular expressions, as their docs provide:
```rb
allow do
  origins "localhost:3000", "127.0.0.1:3000",
          /\Ahttp:\/\/192\.168\.0\.\d{1,3}(:\d+)?\z/
          # regular expressions can be used here
end
```
But, what if you want to set this per environment?

## Restricting CORS Origins via Environment Variable
Set your Environment Variable, CORS_ORIGIN or something similar, and list your origins as a string separated by a comma, or another separator of your choosing.

```env
CORS_ORIGIN="localhost:3000,https://my.staging-app.com"
```
And then fetch your ENV variable and split on your separator:

```rb
allow do
  # this will split to ["localhost:3000", "https://my.staging-app.com"]
  origins ENV.fetch("CORS_ORIGIN","").split(",")
end
```

### Mixing Regular Expressions and Strings
I recently had a need to integrate with the [Netlify](https://www.netlify.com/) deploy preview apps for testing. Their app URLs can be prefixed with a random hash of 24 characters or with a deploy preview of a specific Pull Request ID (`deploy-preview-#{PR_number}`). [Edit: Netlify has changed their permalink structure. Update your regex accordingly.]

Lets say my instance on Netlify is moshertech.netlify.app.

The regex for the random hash looks like:

```
\Ahttps:\/\/\w{24}--moshertech.netlify.app\z
```
The regex for the deploy preview looks like:

```
\Ahttps:\/\/deploy-preview-[0-9]{1,4}--moshertech.netlify.app\z
```
I restricted the PR number to be between 1 and 4 digits, since that covers the first 9,999 PRs. If you have a long running repo and need more, you can loosen the requirement to include a 5th digit.

> Note: the \A and the \z are to denote the start and end of the matching string. This ensures that you are matching against the exact criteria and correctly block other origins that are similar.

In Ruby, the following statements are true:

- a Regular Expression is represented by a Regexp class
- an ENV variable is, by default, a String

Using these two approaches, I was able to set my environment variable to:

```env
CORS_ORIGIN="localhost:3000,\Ahttps:\/\/deploy-preview-[0-9]{1,4}--moshertech.netlify.app\z,\Ahttps:\/\/\w{24}--moshertech.netlify.app\z"
```

And then I was able to set my CORS origins to in this manner, mixing Regular Expressions and Strings from one string Environment Variable:

```rb
allow do
  origins ENV.fetch("CORS_ORIGIN","")
    .split(",")
    .map { |origin| origin[0] == "\\" ? Regexp.new(origin) : origin }
end
```

Is this the best way to do it? No idea, probably not. But it allowed us to be flexible with Netlify’s url format while continuing to block other, non-matching origins. And I felt that it beat allowing everything through the "*" wildcard.

## But Laura, can’t you just use the wildcard in your URL?
Okay first, please watch my [talk](https://www.youtube.com/watch?v=uT1ovJqIjt4) on dropping the Nots and the Justs and why this type of language is problematic.

And no, as far as I could tell, the wildcard `"*"` is meant to allow all domains, and nothing else. Which means, we unfortunately, are unable to use it to simplify our matching style.