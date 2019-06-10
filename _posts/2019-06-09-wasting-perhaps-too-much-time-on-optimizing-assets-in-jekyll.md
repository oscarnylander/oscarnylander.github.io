---
layout: post
---
# Wasting perhaps too much time on optimizing assets in Jekyll

\- But having a little fun in the process

{{ page.date | date_to_string }}

Due to a severe lack of things to do while my partner studies for her final exam for the semester,
I decided to take a stab at making a personal web site. It started out as a very simple project,
consisting of one HTML file, one CSS file, a font loaded from Google Fonts and some icons from
Font Awesome. The page was to be hosted on Github Pages, mostly because I had never tried it out.
To develop, I used one of the neat aliases I've accumulated over the years: `serve`.
It looks like this:

```bash
alias serve="python3 -m http.server"
```

Running it causes `$(pwd)` to be served on `http://localhost:8000`. A simple trick that sometimes
comes in handy.

Finally, it needed a domain, and voila: [oscarnylander.dev](https://www.oscarnylander.dev) was born.

## The problem

Now, being a developer, I could of course not stop myself there. The page seemed to load fine and
all, but I noticed when inspecting that my assets were naturally not minified. So, as always, I
decided to see if someone else had a satisfying solution to the problem at hand: minifying un-minified
code served on a Github Page (without having to check in the minified code).

From that research I learned a few things:

1. Github Pages runs on Jekyll. I had heard about Jekyll before, I knew it was a static site generator
   written in Ruby, but I did not know much more.
2. Github Pages runs a restricted version of Jekyll which does not allow you access to all the plugins
   that could normally be used to modify the behaviour of Jekyll. This could prove to be problematic.

Suggestions to remedy the problem of not having access to Jekyll plugins included having a separate
repository where the underlying code lived and then pushing the artifacts of the build to the
repository that served the Github Page. This seemed a rather inelegant solution, and for the very
slight benefit of serving marginally smaller assets one that I would prefer not to use.

## The HTML

After searching a bit more I stumbled upon
[jekyll-compress-html](https://github.com/penibelst/jekyll-compress-html), a solution written purely
in the templating language employed by Jekyll: Liquid. Since this solution did not require any
external plugins to be employed, it could be used without any problems with Github Pages.
Fantastic! I added it right away.

Integration was straight forward: you simply had to download the
Liquid template, and use it as a layout for what you actually wanted to serve. Sort of like a filter,
if you will.

This did of course require me to refactor the site to a Jekyll-like structure and install
the Github Pages-variant of Jekyll locally though. This was not that challenging, Github Pages does
offer decent documentation on how to do this.

The out-of-the-box settings were mostly good, but I noticed that not all of the spaces
that were used for indentation were completely stripped. I guess they could in some cases carry some
significance. Solving this was fairly simple, though. I simply added the following section in the
Jekyll `_config.yml`:

```yaml
compress_html:
  clippings: all
```

And the white space was gone. Superb!

## The CSS

Now, on to the CSS!

Since there was essentially no CSS included on the site and as I had already set up Jekyll, I
figured I might as well inline the CSS. This could be achieved by placing the CSS in an
`_includes`-directory and then including it when building the page like this:

{% raw %}

```liquid
<style>{% include style.css %}</style>
```

{% endraw %}

`jekyll-html-compress` did take care of some of the minifying of the inlined styles,
but not all. There were still a few white spaces that were not needed,
and that simply would not do! So I went looking for alternative solutions. I then learned
that Jekyll has out-of-the-box support for Sass, and can minify the Sass files it
processes. Fantastic! As SCSS is a superset of CSS, I could then rename the CSS file from
`style.css` to `style.scss` and update the inlining snippet to be the following:

{% raw %}

```liquid
{% capture inline_css %}
    {% include style.scss %}
{% endcapture %}
{{ inline_css | scssify }}
```

{% endraw %}

and adding the following section to the `_config.yml`:

```yaml
sass:
  style: compressed
```

And all of a sudden, there was no longer any white space in the inlined CSS. Marvelous!

## The Payoff

As I did not perform proper measurements of the size right before and right after
enabling all the fixes, it's not entirely straight forward to say how much impact
these steps actually had on the final result, or if it would have mattered at all
post-gzipping, which Github Pages does enable by default. However, I did test the
page in Lighthouse in the Chrome Dev Tools, and can proudly say that my page has
100 in all categories (except for PWA which I don't necessarily care about). Nice!
