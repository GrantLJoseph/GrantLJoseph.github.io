---
layout: post
title:  "Preparing to Apply to Google Summer of Code"
date:   2025-03-25 03:57:00 -0400
category: Low-Level Programming
---
# Introduction
I've wanted for years to apply to Google Summer of Code, an internship program sponsored by Google but run by open-source projects. This year, following my starting college last year, I decided this was the right time. At first, the Tor Project's [proposal](https://gitlab.torproject.org/tpo/team/-/wikis/GSoC#1-project-rewrite-metrics-lib-in-rust) for rewriting a telemetry tool in Rust grabbed my interest, but I decided my Rust experience, while not insignificant, is probably not ready for that task quite yet. Given I'm a cybersecurity major, I wanted something relevant to my career. Rust made sense, being a language essentially designed around security. Continuing my search, I wanted to find another relevant project to apply for. Eventually I found it in the GNOME Foundation's [internship](https://gitlab.gnome.org/Teams/internship/project-ideas/-/issues/58) for developing a proof of concept for security enhancements to [GNOME Papers](https://apps.gnome.org/Papers/). This blog post chronicles my experience applying for the position, up to the point of submitting my application.

# Preparatory Bootcamp
After discovering a different GNOME GSoC internship, I emailed [Jonathan Blandford](https://gitlab.gnome.org/jrb) with some questions. He was very helpful and welcoming, and pointed me towards an online bootcamp happening in only 7 hours. What luck! Except for the fact that it was 5 AM and I still hadn't slept yet. Oh well. I was a bit tired but attended through the power of caffeine. The host and mentors were incredibly helpful and probably spent more time answering my questions than everyone's else's combined. I left with a lot of advice on how to construct my application and what work I should be doing in the meantime to boost my chances of acceptance, including starting this blog. Many thanks to the host [Pedro Sader Azevedo](https://mastodon.social/@pesader) for pointing me in the right direction for setting up this blog using [GitHub Pages](https://pages.github.com/).

# Building GNOME Papers
If I'm going to apply for an internship working on GNOME Papers, a logical first step is making sure I can build and run the app. I cloned it from GitHub, brushed up on the Meson build system, and tried to build it:
```bash
gtk| Dependency wayland-client found: NO found 1.22.0 but need: '>= 1.23.0' (cached)

subprojects/gtk/meson.build:540:19: ERROR: Dependency 'wayland-client' is required but not found.
```

Apparently Ubuntu 24.04 doesn't have the a new enough `wayland-client` version. This isn't supposed to be a problem, though, since Meson is supposed to build from source anything my host system can't adequately provide. After a bit of troubleshooting with the help of [Markus Göllnitz](https://gitlab.gnome.org/camelCaseNick), I was advised to try building the Flatpak version instead of the standard ELF. I've been using Flatpak applications for a long time but I'd never tried to build one before. After researching the process for a few hours, I managed to build Papers successfully! 

# GNOME Builder
I hit a few more roadblocks with the Papers's Flatpak. My changes weren't being reflected in the build, meaning it was always building the original upstream codebase, ignoring my changes entirely. It turns out that's a feature of `flatpak-builder`, the application used to build Flatpak apps. Below is a relevant snippet from the manifest for Papers:
```json
"sources": [
    {
        "type": "git",
        "branch": "main",
        "url": "https://gitlab.gnome.org/GNOME/Incubator/papers.git"
    }
]
```

Despite the fact that I was literally in a local copy of the Papers repository, `flatpak-builder` will always clone from GitLab for the build. I rewrote it to build using the local copy and everything finally built and ran as expected.
```json
"sources": [
    {
        "type": "dir",
        "path": "/home/grant/src/papers"
    }
]
```

The issues I hit with Flatpak lead Markus to recommend I switch to [GNOME Builder](https://apps.gnome.org/Builder/), an IDE built specifically for developing GNOME apps. It makes setting up a development environment painless and I've generally liked my experiences with it thus far. It handles building and running the Flatpak for Papers in one click. With my development environment ready to go, I was ready to search for a starting contribution to make to include in my GSoC application.

# GNOME Apostrophe
I've written this blog so far using [GNOME Apostrophe](https://apps.gnome.org/Apostrophe/), a genuinely excellent Markdown editor. After looking through its issue tracker, I found [this issue](https://gitlab.gnome.org/World/apostrophe/-/issues/506). I needed to fix the export file dialog so that it defaulted to opening the parent folder of the currently open file. I found the code responsible for creating the GTK file chooser dialog and added code to set it's starting directory.
```py
self.dialog = Gtk.FileChooserNative.new(
              _("Export"),
              None,
              Gtk.FileChooserAction.SAVE,
              _("Export to %s") %
              self.formats[self.format]["name"],
              _("Cancel"))

# Shortened for brevity

# / is the base path when the current file has not been saved
if file.base_path != "/":
    self.dialog.set_current_folder(Gio.File.new_for_path(file.base_path))
```

# Back to GNOME Papers
With my patch submitted to Apostrophe, I went back to Papers to try and fix an issue there. I started with Apostrophe first because its codebase was generally easier to work with. My eye was drawn to an [issue](https://gitlab.gnome.org/GNOME/Incubator/papers/-/issues/398#note_2390037) with annotation timestamps. The timestamps were always showing in UTC and not the device's local time. I added code in the relevant function to parse the existing date/time string and generate a new one, this time respecting the device's timezone and 12/24 hour clock setting.
```rust
let Some(modified_original) = annot.modified() else {
    return format!("<span weight=\"bold\">{}</span>", label);
};

let Some(modified) = modified_original.strip_suffix(" UTC") else {
    return format!("<span weight=\"bold\">{label}</span>\n{modified_original}");
};

let modified = modified.to_string() + "+0000";

let Ok(modified_utc) = chrono::DateTime::parse_from_str(&modified, "%a %d %b %Y %H:%M:%S %p %z") else {
    return format!("<span weight=\"bold\">{label}</span>\n{modified_original}");
};

let format_string = if is_12_hour_clock() {
    "%a %d %b %Y %I:%M:%S %p"
} else {
    "%a %d %b %Y %H:%M:%S"
};

let modified_local = modified_utc.with_timezone(&chrono::Local).format(format_string);
format!("<span weight=\"bold\">{label}</span>\n{modified_local}")
```

# Application
That brings us to where I am today. My application will be submitted tomorrow at the time of writing. I am very grateful to everyone who helped me get here. I look forward to hopefully seeing much more of my new acquaintances at GNOME this summer.