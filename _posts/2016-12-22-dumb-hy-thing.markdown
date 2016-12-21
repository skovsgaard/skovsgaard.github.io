---
layout: post
title:  "A Dumb Thing To Do With Hy"
date:   2016-12-21 22:01:00 +0100
tags: [lisp, hy, python, web]
---
Hy is a fun little language. In its beautiful silliness it's a Lisp frontend to Python. What this means is that you can write beautiful code with all the speed and concurrency of Python and all the readability of Scheme. In other words, Hy serves no purpose except being oddly entertaining and strangely pleasant to work with (for us parenthesis admirers).

Now, let's try something fun to get acquainted with Hy. We'll make a relatively dumb web app which just presents you with a form and then tells you what you just posted using that form. We'll do that using the Python framework called Tornado, ostensibly made for fast, asynchronous web servers, but we'll just use it for a simple web thing for now. To get started, find a directory you like and create a virtual environment there. You might do it something like `virtualenv whyrlwind` because Hy loves working in "hy" in every name ever.

Now, once you've activated that virtualenv, run `pip install hy tornado`. Then, create a file called `server.hy` and open it up in your favorite editor which is Emacs. The Hy plugin for Vim is sorely broken and no one bothered fixing it because your favorite editor is Emacs.

We'll want two routes in our tiny application. One to get a form and another to post that form and reply back to you with what you said to it. First though, you'll want to import tornado in your Hy application:

```hy
(import tornado.ioloop
        tornado.web)
```

That first line imports the main event loop of Tornado, basically the guts of what'll be our little app. The second line imports the classes we need to be making web handlers out of in a short second.

You're actually ready to start writing the handler now so literally just the one second:

```hy
(defclass SillyHandler [tornado.web.RequestHandler]
  [[get-hello (fn [self]
                (self.write "<form action='' method='post'><input type='text' name='silly-input'><input type='submit' value='yeah i said it!'></form>"))]
   [post-hello (fn [self]
                 (self.write (+ "You said " (self.get-body-argument "silly-input"))))]])
```

BAM! You just wrote your whole application. Well, almost, because Hy is weird about certain things that Python will allow. For one thing, Hy won't allow you to overwrite built in methods of classes (like `get`) and unfortunately this is exactly what Tornado needs to do its magic. Therefore, we'll add these two ugly lines immediately after the beautiful handler we just wrote:

```hy
(setv SillyHandler.get SillyHandler.get-hello)
(setv SillyHandler.post SillyHandler.post-hello)
```

Yeah, not very pretty when we're defying the gods like that. Oh well, let's keep going. Tornado still needs to know a few things before we can make it serve our neat handler. For instance it needs to know what route to listen for. We only have so much functionality, so let's just make it listen for `"/"`:

```hy
(defn make-app [] (tornado.web.Application [(, r"/" SillyHandler)]))
```

That bears explaining. We are creating a function that we'll invoke in a minute and in that function we're creating an instance of Tornado's `Application` object with a list of one argument which is the tuple containing our base route and the name of the handler we want to be responsible for that route. Next up we need to actually run this whole thing:

```hy
(defmain [&rest args]
  (let [[app (make-app)]]
    (do (app.listen 8888)
        (.start (tornado.ioloop.IOLoop.current)))))
```

Hy has this neat shorthand for the Python idiom of `if __name__ == "__main__"` which is aptly called `defmain`. In this `defmain` call we're binding the result of calling our exciting `make-app` function to `app` and then making that app listen on port 8888. Lastly we're starting up the whole thing by telling Tornado to fire up its event loop and then magic happens and you can go to `localhost:8888` and insult a web server. All by the magic of Hy.

If you enjoyed this, you really ought to check out www.hylang.org for more hylarious shenanigans, most of which are way better ideas than this one. Another place to find said shenanigans is at github.com/hylang/hy. If you hated this waste of time, you can come scream at me via text at twitter.com/status402.