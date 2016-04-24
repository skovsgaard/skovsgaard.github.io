---
layout: post
Title:  "Nicer font rendering in GNOME Web/Epiphany"
date:   2016-04-25 00:00:01 +0100
categories: jekyll update fedora
---

If you've gotten tired of the massive memory consumption of big browsers or simply don't need your browser to do a whole lot except show web pages, you may have tried GNOME Web (the browser formerly known as Epiphany).

If that is the case, odds are you've run into some ugly fonts which don't follow the font rendering rules you've set in Gnome Tweak Tool or via GSettings. This is because of Web's inexplicable decision to not abide by the central GNOME configuration but rather rely on the settings made by fontconfig.

While this isn't obvious, there's a relatively easy fix to the problem.

If it doesn't exist, create a local config file with the command, `sudo touch /etc/fonts/local.conf`\*, open it with `nano` or `vi` and place the following XML content in it:

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <match target="font">
    <edit name="hintstyle" mode="assign">
      <const>hintslight</const>
    </edit>
  </match>
</fontconfig>
```

Now, if the file already existed, and you just wanted to add information on how hinting should work, just place the `<match>` block inside the `<fontconfig>` tag of the existing file.

And that's it. You are done and only need to run `fc-cache -f` for your changes to take effect and yield a visible improvement in Gnome Web as well as in other applications which rely on fontconfig. Naturally, you're not forced to use `hintslight`, if you prefer more or less rounded fonts. Other options include `hintnone`, `hintmedium`, and `hintfull`.

**Note:**

\* `/etc/fonts/local.conf` applies your change across user accounts on a machine. In case you want to apply these changes to just your user account or simply want a config file that's easier to back up, use the file, `~/.config/fontconfig/fonts.conf` instead.
