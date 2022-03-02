---
title: "Bug fixing without source code: How to reverse engineer android apps"
layout: post
description: "A walkthrough on how I triaged a bug in an app I use, using android app reverse engineering tools"
---

# Bug fixing without source code: How to reverse engineer android apps

{{ page.date | date_to_string }}

I'm generally not a big fan of social media. One exception I make to that rule, however,
is HackerNews, a fantastic source of news and discussion for people with similar
interests to me. Like everyone else, I've moved towards using my phone more and more
for all my computer-related interactions, and browsing HackerNews is no exception.
When on my phone, I generally prefer using native applications over web pages. Hence,
I started using the excellent HackerNews client called
[Harmonic](https://play.google.com/store/apps/details?id=com.simon.harmonichackernews).

## The bug
Harmonic is generally a very good HackerNews-client, almost as good as a
HackerNews-client can be, given their API. There was one pretty major bug
that did bother me, though: When typing a comment, if the comment was long
enough to fall behind the keyboard, then subsequent lines fell behind the keyboard.
Suffice it to say, this is not the intended behaviour for this type of
interaction on android - most often, the text input should only have the
same height as the window of the app minus the keyboard, and become scrollable
when the content becomes larger than the text input is.

I lived with this bug best I could for many months. As I don't comment too
often, it was not something I struggled with too much, but finally I felt
like I had enough. Luckily, as I've worked with developing android apps
at my day job for quite some time now, I could recognize the symptoms of this
issue. I developed a first working theory and contacted Harmonics developer,
but I felt like this was not quite enough - I wanted to validate the theory,
in the way I would be able to had I had access to the source code. Given my
knowledge of android applications, I felt like I would be able to atleast
validate my working theory without too much work.

## A quick aside into android application components
Let's quickly walk through what an android application actually is.
That is to say, the artifact that gets downloaded to the android device,
and then executed, what does it consist of?

### The APK
The APK (Android application PacKage) is not much more than a ZIP-archive
containing some built code and assets. You can actually verify this by
running `unzip ${ANY_APK}.apk`, and find that it works without any issues.

There are a few more details to be known - all APKs must have a signature,
and they must also be aligned, so any ZIP-archive wont do.

Once unzipped, we find the contents that make up the android application:

### The Manifest
The Manifest (`AndroidManifest.xml`) is a listing of what the application
contains in many ways. Any activities that are used must be listed here,
and this file also contains what permissions the app wants to use, any
links that it wants to be able to open, and so on. In an APK, this file
is not human-readable, but in a binary encoding.

### Resources
Resources in android are also most often XML-files, and they will also
be in a binary encoding in the APK.

### .dex-file
The .dex-file contains the built bytecode, and is what gets executed
as a part of the installed application. These are obivously not human-
readable either.

Given the large amount of non-human readable components in an APK, we
need some tools to help us reverse engineer the application.

## Reverse Engineering-Tools
Along the way, I found some very useful tools to help me reverse engineer
the application.

### `apktool`
This was the cornerstone reverse engineering tool I made use of. It takes
an APK, extracts it, and decodes the resource files into human-readable
format again. Additionally, it takes the .dex-file and decompiles it into
`smali`, a more human-readable representation of the bytecode.

You can read more about `apktool` [here](https://ibotpeaches.github.io/Apktool/).

### `objection`
`objection` is a reverse engineering toolkit for android. It wraps `apktool`
and some other useful things, in order to simplify the process of modifying
and introspecting a running android application.

I used `objection` primarly to make the APK debuggable, but also to
streamline the process of modifying and re-building the APK, when I had
something that I wanted to try.

Another fun feature, and arguably the primary purpose of `objection`, is
that it injects `frida` into the APK. Frida then allows us to do some
runtime introspection of the application. I didn't utilize this capability
that much, but it is a noteworthy part of the tool.

You can read more about `objection` [here](https://github.com/sensepost/objection)
and `frida` [here](https://frida.re/).

### smalidea
smalidea is a plugin for IDEA-based IDEs that provides syntax highlighting
and more. This plugin made debugging the application much simpler.

You can read more about smalidea [here](https://github.com/JesusFreke/smalidea).

### Honorary mention: `APKLab`
APKLab is an extension for Visual Studio Code, allowing for a quicker
workflow of modifying and rebuilding APKs. I didn't quite get this
tool to work, and I found it at the very end of this process, but it
seemed like it would be a better fit than the workflow I adopted for
doing this type of work. Hence the honorary mention.

You can read more about `APKLab` [here](https://github.com/APKLab/APKLab).

## First theory: wrong `windowSoftInputMode`
My first theory was simple: What if the author simply forgot to
add the `adjustResize` attribute to the Activity? This theory was fairly
straight-forward to validate - this attribute is added to the previously
mentioned `AndroidManifest`, and hence I would be able to verify it
quite quickly.

For reference, `adjustResize` is one possible mode that can be set on
an Activity's `windowSoftInputMode`: having it set makes it so that when
the keyboard shows, the activity adjusts by resizing its window, making
the new application window smaller in order to accomodate the keyboard.

The first step was to download the APK from the device I had it on:

```bash
$ adb shell pm list packages -f -3 | grep harmonic
```

This command listed where the APK was located on my device, which I could
then use as input to the second command:


```bash
$ adb pull /data/app/.../com.simon.harmonichackernews-.../base.apk
```

and I was then presented with the apk in the current working directory.

Then onto decompiling the APK:

```bash
$ apktool d base.apk
```

which extracted the APK, decoded any resources and decompiled the .dex-
file down to .smali-files.

Then it was simply a matter of looking into the `AndroidManifest`: was
it set to the correct mode?

As it turns out, yes. The correct mode was indeed set on the Activity. Darn.
No matter - given how far I had come, I was not about to give up - I had
a few more possible theories I could try out.

## Second theory: improperly configured `WindowInset`s
My second theory was that there was some form of misconfiguration with
regards to Window Insets. During my work, this was something I remember
that I often used to stumble into, and small mistakes could lead to this
type of bug.

In order to actually understand whether this was the problem or not,
I had to get down and read some of the decompiled bytecode in smali-format.

The best way I found to do that was to use the Android Studio-function called
'Profile or Debug APK' (Found under File -> Profile or Debug APK). Combined
with smalidea, it was quite straight-forward to set breakpoints and inspect
the executing code.

Given that the code was obfuscated, and smali is not quite as readable as
regular Java or Kotlin, I had to do some gradual transcription of the code
as I stepped through it, but I managed to discern that there was no use of
the `WindowInset`s-API in this app. I did, however, find some suspicious
method calls, that was my next stop in the investigation.

## Finding what was actually wrong
Early on in the Activity that allowed for replying to a comment or post,
there was a method call, which after a while called
`View::setUiSystemVisibility`, with what I could finally figure out
was `View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION`. A few quick searches
indicated that doing this could interfere with the proper function of
`adjustResize`.

All I had to do at this point was to remove this method
call and see if the problem persisted. I decompiled the APK, removed this
particular method call, and recompiled it again, and tried opening the
keyboard while in the offending activity. And voilà, the UI did resize,
and it was finally possible to input as much text as I wanted, without
having any of it be obscured by the keyboard.

It was by no means perfect - as the app had been designed with
`View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION` in mind, the insets were
now incorrect, but as I had identified the root cause, I felt satisfied,
and stopped at this point.

I've contacted the author of the app about the root cause, and offered
to write a fix myself, should I get access to the source code.
