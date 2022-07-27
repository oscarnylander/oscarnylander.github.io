---
title: "Overcoming a podcast app-bug using Cloudflare Workers"
layout: post
description: "How I circumvented a bug in a podcast app, by building a small serverless function on Cloudflare Workers"
---

# Overcoming a podcast app-bug using Cloudflare Workers

{{ page.date | date_to_string }}

I listen to a lot of podcasts, mostly when doing chores or exercising, to overcome the boredom that these
actions can bring. I mostly listen to podcasts on RadioPublic, which I chose a few years back. Due to
sunk cost, I have not been able to bring myself to look for an alternative, despite RadioPublic seeming
to not be in active development anymore, having been updated last around a year ago.

RadioPublic allows the user to listen to any podcasts listed in their application, but also to listen
to any podcast which provides an RSS-feed. This functionality is, somewhat unintuitively, provided
by simply pasting an RSS-feed URL in the search bar, at which point the podcast provided by the RSS-feed
will show up as a search result. However - there's a bug in this functionality.

## The bug

As mentioned in the previous section, the "Add podcast by RSS-URL"-feature is somewhat unintuitive. It
also has a bug - it only seems to actually respond to URLs that have a file extension, with this file
extension being either `.xml` or `.rss`. Note that I haven't actually decompiled the application to
verify that this is what it does - I merely deduced this by observation. Should the URL not have a
file extension ending with `.xml` or `.rss`, the URL will simply be interpreted as a search query in
the search bar, which sometimes coincidentally shows the right result, but not necessarily.

This became an issue for me when I wanted to listen to a podcast hosted by Libsyn - a podcast host
which does not provide their RSS-URLs with a file extension - they merely provide a link ending in `/rss`,

There could be some discussion on which program is incorrect here, but I am going to argue that it's
definitely RadioPublic not doing the right thing here - file extensions are not my preferred method
of conveying file contents. I think that a more correct approach would be to inspect the `Content-Type`-
header of the HTTP response of the URL, and use that as the basis for whether the URL is a podcast-feed
or not. Some quick CURLing to the podcast URLs from Libsyn shows that it does indeed have the correct
`Content-Type`-set on their HTTP responses, when requesting this URL.

## Potential ways to cope with the issue

I was a bit frustrated, as I had found a very interesting podcast episode to listen to, that was
hosted on Libsyn. The way I saw it, I had a few ways to go about it:

### I could listen to the podcast episode in their web-based player

I did not like this method, as I potentially wanted to be able to download the episode and track
my listening progress, and potentially listen to more episodes from this podcast.

### I could download the MP3-file containing the podcast episode, and listen to it in another media player

I didn't find this method appealing either, as I would probably not be able to track my listening
progress in a good manner. I could maybe have used an audiobook-app to listen to the podcast-episode,
but this did feel a bit like accepting defeat.

### I could switch podcast-apps altogether

I must admit that I found this option a bit tempting. However, having tracked all my listening progress
in RadioPublic, and being very fluent with the UI of RadioPublic, I had a bit of cost sunk into
continuing to use RadioPublic. I might switch some day, though, given the apparent inactivity
of RadioPublics development team.

### I could make a server that simply served the links from Libsyn, but appended `.xml` to the end of the URL

This seemed like a fun way to tackle the problem, and a good opportunity to practice my Rust skills.
I initially went with this approach, but after a while I kind of got bored with the prospect of having
to actually host this server, and acquire a TLS certificate for it. It just simply seemed to be too much
effort for such a small workaround.

### I could do the exact same thing as previously mentioned, but use a serverless solution instead

Finally, a solution that seemed just reasonable enough to work. I had read about Cloudflare Workers
before - that they have a generous free tier, and that they support Rust out of the box. Combine
this with me already using Cloudflare for this blog, and it seemed like a really good match.

## Getting up to speed with Cloudflare Worker development

Getting all of the tooling in place for developing Cloudflare Workers was surprisingly easy.

First, I installed the CLI-tool `wrangler`, to facilitate development:

```sh
$ cargo install wrangler
```

Then, I created a project from a template:

```sh
$ wrangler generate libsyn-mirror-cloudflare-worker --type="rust" \
  --template="https://github.com/cloudflare/rustwasm-worker-template"
```

Wrangler lets you run your worker locally for development in a very simple manner:

```sh
$ cd libsyn-mirror-cloudflare-worker
$ wrangler dev
```

Which then lets you shoot requests to the worker as if it were hosted, but on your local machine, on `http://localhost:8787`.

## The code itself

Cloudflare workers in Rust use a small router, on which you can register URL patterns that get responded to by calls to a
provided function.

My idea was as follows:

1. Gather the Libsyn podcast ID from the URL as a path parameter
2. Using the ID, construct the URL to the Libsyn-RSS feed (this can be derived using only the ID)
3. Respond to the request with a 301 redirect to the Libsyn-RSS feed

The implementation was extremely simple:

```rust
// ...
router.
    get("/:podcast_id/rss.xml", |_req, ctx| {
        // Get the podcast ID from the URL parameter
        let podcast_id = ctx.param("podcast_id");
        // Return Bad Request if it was not provided
        if podcast_id.is_none() {
            return Respons::error("Missing parameter `podcast_id`", 400);
        }
        // Safe to do, as we checked that it's not none before
        let podcast_id = podcast_id.unwrap();
        // Construct the URL
        let url = Url::parse(&format!("https://sites.libsyn.com/{}/rss", podcast_id));
        // Return Internal Server Error if URL parsing fails
        if url.is_err() {
            return Response::error("Could not parse redirect URL", 500);
        }
        // Safe to do, as we checked that it's not err before
        let url = url.unwrap();
        // Implicit return - we return a redirect with status 301: Moved Permanently
        Response::redirect_with_status(url, 301)
    })
// ...
```

The comments should provide a good level of understanding of what the code does, even for readers
who are not very-well versed in Rust.

## Deploying and testing it out

Deploying Cloudflare workers is also very simple, using `wrangler`:

```sh
$ wrangler publish
```

I was now provided with a URL to the worker, and could test it out in RadioPublic. And lo and behold,
it worked - I could now load the podcast I wanted to listen to. I could finally declare victory, and
rest easy, having circumvented the bug.

## Conclusions

Cloudflare workers seem like a really good option for certain tasks. They are highly performant, and
very simple to get up to speed with. They will probably be high up on the list of tools I choose for
future personal projects. Rust continues to be a very pleasant language I wish I would be able to
use more professionally.
