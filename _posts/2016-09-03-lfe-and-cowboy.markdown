---
layout: post
title:  "A tiny HTTP server using LFE and Cowboy"
date:   2016-09-03 20:06:00 +0100
categories: jekyll update
tags: [lisp, flavored, erlang, lfe, cowboy]
---

This tutorial will cover how to get the simplest possible Cowboy server up and running using [Lisp Flavored Erlang](http://lfe.io). Why would you want that, you ask? First and foremost because learning is fun and Lisp is a great way to get to thinking about how you think. However, some parts of Erlang are particularly well suited for being written as Lisp code, as we'll see here. This guide tries to assume as little as possible, so if you're new to Erlang, come yell at me on Twitter if this gets too complicated.

The easiest way to get a basic scaffold for an OTP application with LFE is to install [arpunk's LFE templates for rebar3](https://github.com/arpunk/rebar3_lfe_templates). Follow the instructions in the README to get started.

Once you've got that set up, go to your usual project directory and run the following command:

    $ rebar3 new lfe_app name=cowboy-coding

Feel free to peek at the files created in the `src` directory, which is mostly where we'll be working. What the rebar3 template, `lfe_app` created for us is a base OTP app with a single supervisor and a single gen_server.

Now, with that out of the way, open `rebar.config` and find the section titled `deps` at the top of the file. Add add the following to the beginning of that list:

    {cowboy, "1.0.4"},

Remember the comma. Because the `rebar.config` file is just full of Erlang data, it's pretty specific about punctuation.

Now all you need to do to fetch Cowboy from Hex.pm is to run `rebar3 deps`.

The only thing you need now, before we can start coding is to add `cowboy` to the file called `cowboy-coding.app.src` in the `src` directory. The `applications` section of that file should now look like this:

```erlang
{applications, [
    kernel,
    stdlib,
    cowboy
]}
```

Again, adding the comma after `stdlib` is important.

## Setting up a router

To get started, open the file called `cowboy-coding-app.lfe` located in the `src` directory. In here, you'll see a couple of functions that are defined for you but don't do much as is. One is called `start` and the other `stop`; pretty useful pieces to have when building an application.

Starting up a Cowboy server is done by a call to the function `cowboy:start_http` but it needs the right data to serve traffic correctly. And to make it a little bit easier to reason about that data, let's break a bit of it off into a separate function. Let's define the following function at the bottom of `cowboy-coding-app.lfe`:

```
(defun router-match-spec ()
  "Returns the match spec we will be using for our router"
  '(#(_ (#("/" cowboy-coding-handler ())))))
```

This is all very confusing and begging for some explanation. So here's - line by line - what's going on:

1. As you may have deduced by now, `defun` is the LFE way of saying "We'd like to define a function". The keyword is borrowed from Common Lisp, but semantically it's identical to Elixir's `def` or in other words: What it defines is just an Erlang function. While we're on the first line, it's worth noting that in the Lisp style, LFE functions are dashed, unlike the snake_cased functions in Erlang. This is particularly important when you need to interoperate with Erlang or Elixir, which won't necessarily be happy with the classic feel of LFE.

2. Why is there suddenly a string in the middle of everything? Well, if you're comfortable with either Elixir, Python, or any of a number of Lisp dialects, you may recognize it as a docstring. If not, this one string before the body of the function can be described as a small unit of documentation, embedded into the function.

3. Now it's getting weird. What's with all the quotes? What is everything here? Well, in Lisp - because code is data and vice versa - a list may look almost identical to a function call, which is particularly apt in this example. We start out with a quote to signify that this is a list literal. Everything in after the first block is taken literally and not as the result of a function, unless we tell it otherwise by using comma to signify "unquote" (spoilers: we'll try this later). For now though, we will just define an Erlang match specification which really deserves a small explanation before we continue.

Match Specifications|
--------------------|
An [Erlang match specification](http://erlang.org/doc/apps/erts/match_spec.html) is defined as "a small 'program' that will try to match something" which serves two purposes here: firstly it's how Cowboy lets you define your routers in a very concise format, and secondly it demonstrates that Lisp really shines at tasks where we would prefer to treat data as code.|

With this bit of knowledge covered, let's look at the match specification we're using again:

```
'(#(_ (#("/" cowboy-coding-handler ())))
```

We have a list of one tuple defining our match specification. It's using the Erlang wildcard character, `_` which would normally need to be in single quotes when used from Erlang, but because our data is already quoted (remember, we started with the single quote), we just need to type out an underscore. What this wildcard means is that we want to match on everything. In our case, that means that we want all our requests to go to the same handler, because we're just using this as a learning example. In more serious projects, you may want to use a more fine grained approach with multiple routers. The second element in the tuple defines what we want to do when we match, which is take all requests to the root path and route it to the `cowboy-coding-handler` module.

Before we define that handler module, let's get back to the `start` function I mentioned ages ago, the place where we actually needed the match spec originally:

```
(defun start (_type _args)
  (let* ((dispatch (cowboy_router:compile (router-match-spec)))
         (`#(ok ,_pid)
          (cowboy:start_http
            'lfe_http_listener
            100
            '(#(port 8080))
            `(#(env (#(dispatch ,dispatch)))))))
    (cowboy-coding-sup:start_link)))
```

Like the other function we defined, this begins with a defun. `start` takes two arguments, but since we don't need them, we're prepending those arguments with the wildcard character, `_`.

Next, we're creating a `let*` expression. `let` is Lisp's way of describing local bindings. It allows us to create a block containing the values we need to operate on in our function and only return what we actually need to. What the asterisk after let means is that we're doing a sequential `let`. That is, we want to use variables from one part of the `let` binding in a later one. We're creating a value called `dispatch` which holds the result of calling `cowboy_router:compile` on the match spec we created before. This `compile` function returns a set of dispatch rules, which is why we needed the sequential `let*`.

We want to pass it to `cowboy:start_http` which also takes a name for our listener, the number of acceptors our connection will need, an options tuple in which we're just telling it to listen on port 8080, and lastly, the server's environment which contains our dispatch rules that we just defined.

Finally, in the actual body of the `let*` expression, we can just call `cowboy-coding-sup:start_link` which returns the same as it did when it was generated as part of our scaffold. At this point, you can check whether your app is running by first compiling your project, then starting your app from your LFE shell:

```
$ rebar3 deps
$ rebar3 compile
$ lfe
> (application:ensure_all_started 'cowboy-coding)
#(ok ((ranch crypto cowlib cowboy cowboy-coding)))
```

If this doesn't look the same in your terminal, try to backtrack and see if you missed parentheses, commas or anything else along the way. Worst case scenario, I will put a link to the canonical Cowboy tutorial at the end of this post, so you can go try that out instead.

Now, even if this does work, going to http://localhost:8080 as we've specified will crash the HTTP server process. Why? Well, remember the HTTP handler we didn't actually build yet? Yup, we're trying to call it and crashing as a result, without ever giving a response.

## Creating the HTTP handler

Now, create a file in `src` called `cowboy-coding-handler.lfe`. In this file, start out by adding the following:

```
(defmodule cowboy-coding-handler
  (export all))
```

You may recognize this from the other files. What these lines do is simply tell LFE (and by extension Erlang) that we're defining a new module with the name `cowboy-coding-handler` and that we want to expose every function defined below.

Next it's time for the handler logic itself. Since we're just getting a simple handler up and running, the following should be enough:

```
(defun init (_type req opts)
  `#(ok ,req 'no-state))

(defun handle (req state)
  (let ((`#(ok ,res)
         (cowboy_req:reply
          200
          '(#(#"content-type" #"text/plain"))
          #"Hello, LFE!"
          req)))
    `#(ok ,res still-no-state)))

(defun terminate (_reason _req _state)
  'ok)
```

That looks like a lot, but let's break down what's going on here.

First, we define an `init` function which takes three arguments, request type, request, and options. We don't do a whole lot with it, but simply return a tuple containing `ok`, the request as it came in, and lastly - because we don't really try to keep state on our handler - the atom, `no-state`. You can make this whatever you want, but since handlers are an OTP behaviour just like gen_server or supervisor, you have the option to keep some kind of meaningful state on them.

Next up is the important function, `handle`. This is where the action happens:

We start out by opening a `let` expression, and you'll notice this time it's not the sequential kind but just a regular one (we only need a single value here).

Within the `let` we state that we want `#(ok ,res)` - that is the atom `ok` and then a response - to match the result of calling `cowboy_req:reply` with the arguments needed to respond to a user. Those arguments are a status, `200`, some headers which in our case is just `content-type:text/plain`, most importantly some text to respond with, and lastly the request that came in.

Finally, we can say `#(ok ,res still-no-state)` which returns just that, telling Cowboy that the response is successful and we want to respond with what we just defined `res` as. Like I mentioned earlier, the comma is necessary because we're binding a value to `res` from inside a quoted expression, started by the quasiquote (you may know it as backtick). The atom, `still-no-state` is like with the example in `init` something we don't care about because we don't care about the state here. In a real world HTTP handler, you will need to be more careful about how you treat your state.

Once we're done defining our actual handler logic, we make sure we have a `terminate` function so our handler will know what to do once the show is over. This will get called even if the handler exists normally, so we want to be sure our server knows what to do.

And that's basically it.

Now you can go to your terminal, run `rebar3 compile`, followed by `lfe` and then, when in your LFE shell, `(application:ensure_all_started 'cowboy-coding)` and your app should be running. Then, go to http://localhost:8080 and you should see a message from yourself.

You literally just built a web server and definitely not the easy way. Go you! If your project isn't compiling or otherwise doing something weird, either I made a mistake or you can go [here](https://github.com/skovsgaard/cowboy-coding) to see (or clone) the sample project and check what might've gone wrong.

This tutorial was adapted for LFE and Rebar3 from the official Getting Started guide at the [NineNines site](http://ninenines.eu/docs/en/cowboy/1.0/guide/getting_started/) which uses erlang.mk and regular Erlang. Go check that one out, and make sure to dig around the site in general for more information on how to use Cowboy, which contains handlers for both building REST APIs and Websocket servers as well as the simple HTTP ones we've tried here.
