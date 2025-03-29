---
layout: post
title:  "Fixing Broken Background Permissions When Building Flatpak Apps"
date:   2025-03-29 03:00:00 -0400
---
While preparing my development environment for [GNOME Papers](https://apps.gnome.org/Papers/) while [working on my GSoC application](/gsoc/2025/03/25/preparing-to-apply-to-google-summer-of-code.html), I encountered an issue with `flatpak-builder` that prevented me from building Papers. This post explains what happened and how I fixed it.

# The Problem
When I opened [GNOME Builder](https://apps.gnome.org/Builder/), after having successfully built Papers just a few days prior, it looped between "fetching", "cargo check", and "indexing", forever.

![GNOME Builder screenshot showcasing the issue](/assets/images/2025-03-29-fixing-broken-background-permission-when-building-flatpak-apps/gnome-builder.png)

I assumed this was a bug in Builder so I tried to build from the command line. No matter which branch I tried building, the build always ended like this:
```
[38/245] Building CXX object CMakeFile...er.dir/poppler/FontEncodingTables.cc.oError: module poppler: Child process exited with code 137
```

# The Cause
After some digging, I found out that a 137 exit status generally means the program was killed for running out of memory. I spent a while checking if maybe `flatpak-builder` imposed some kind of memory usage limit before deciding this was not the case. Eventually, I found [this almost 5-year-old GitHub issue](https://github.com/flatpak/xdg-desktop-portal/issues/478) detailing the problem. Despite a 137 exit status usually meaning the program ran out of memory, exit statuses are just convention; any program can return any exit status it wants, for any reason, without enforcement of standards or conventions. [XDG Desktop Portal](https://flatpak.github.io/xdg-desktop-portal/), a D-Bus-based service to allow Flatpak apps to bypass the Flatpak sandbox while remaining within the confines of their granted permissions, was killing the Flatpak app that `flatpak-builder` starts temporarily to contain the build environment. It turns out that I had accidentally revoked the background permission for the development version of Papers. Since the build environment is technically a Flatpak app running without a window, it will be killed just like any other Flatpak app with that permission revoked if it does not create a window within a set amount of time after starting.

# The Solution
Since apps like [Flatseal](https://github.com/tchx84/flatseal) can be used to manage Flatpak permissions in-depth, I should've been able to just re-grant the background permission. The *development* version of Papers wasn't installed, however. GNOME Builder doesn't install development versions of apps system-wide, so Flatseal thought the app was not installed. I had manually installed the development version on accident before, which was probably when I also managed to revoke the permission, before uninstalling the development version. Uninstalling it did not reset the permissions, however, putting me in a situation no GUI would help me solve - I had to reset the still stored permissions of an uninstalled app. I could not just install the development version again because I could not build it.

I needed to manually reset the permission by finding where exactly the system stores Flatpak permissions. My search eventually brought me to the `~/.local/share/flatpak/db` directory which stores a series of GVariant database files containing at least some of the permissions granted or revoked to Flatpak apps. One of them, `background`, was the database I needed to tamper with. I tried editing it with a hex editor but could not make that work. I resigned myself to just deleting the database entirely and hoping that didn't break my system. I anxiously rebooted and... the problem was solved! The database was regenerated, empty. Papers could finally build again, both from the CLI and from within Builder.

# Final Thoughts
This took quite a while to solve, so I hope someone can benefit from my chronicling it. Flatpak needs a system to ensure uninstalled apps have their permissions reset so we can prevent this issue going forward. I'm also curious why Flatseal didn't show the permissions, even though they were still present in the database(s). A mystery for another day perhaps.