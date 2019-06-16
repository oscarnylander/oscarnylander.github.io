---
layout: post
---

# Automating the stuff you never wanted to do anyways

{{ page.date | date_to_string }}

One of my minor hobbies is home automation - I like to automate
some of the aspects of my life. This is the story of how I
'accumulated' my setup.

## Sleepless in Sweden

Since I was a teenager, I've never quite slept well. Maybe it was the
ever-shifting lights of Sweden, Maybe it's the fact that I've spent
as much time as I possibly could in front of a screen, maybe it could
be something else. It has become a significant part of who I am.

In an effort to help me sleep better, a few years ago I decided to
get lights that I could control the brightness of. The idea being
that I could reduce the brightness to as low as possible before going
to bed, and that this would help me fall asleep easier. The lights
I decided to get was IKEA Trådfri. After installing these lights
I noticed that the lights could do much more than just adjusting the
brightness - I could also control the color temperature and do other
rudimentary home automation things through the IKEA Trådfri app.

I found these functions to be quite useful, and I got a taste for
home automation in general. My favorite function in the IKEA Trådfri
app was to turn on the lights automatically in the morning, to help
simulate a sunrise - this is quite useful during the winter months
in Sweden as the sun does not necessarily rise before you leave
for work.

## Keeping it clean

I've never quite liked vacuuming. I grew up in a fairly big house,
and vacuuming was a time-consuming chore. Admittedly, I was quite
lazy growing up (and maybe still am, to an extent). This distaste
for vacuuming has stuck with my as I moved out. One evening while
I was visiting my cousins, one of my cousins was talking about
robot vacuums. He specifically recommended the Xiaomi RoboRock S5,
which he had heard a lot of good things about. Despite the
fairly steep price of around 5000 SEK, I decided that never having
to vacuum the floor myself again would be worth it.

I was completely hooked after a few days of using it. I figured
that instead of the fairly infrequent schedule of ignoring the
buildup of dust in my home, I could just run the robot vacuum as
soon as I felt like it. This turned in to more and more times I
used the vacuum, until I settled on a fairly regular schedule of
turning on the vacuum just before I left for work, and running
the mop function on weekends/when I remembered to do it.

## Let there be light

After a few happy months of using my IKEA Trådfri lights, the
battery ran out on the remote control, which I had mounted
right next to the door, so that I could use it to turn off all
the lights just as I left for work in the morning, and so that
I could turn the lights on when I arrived home from work.

Now, I could probably just have gone down to the local supermarket,
bought myself another battery and called it a day, but I decided
that what I actually wanted was not to use the button every day
when I left my home for work; what I actually wanted was for
the lights to be turned off while no-one was at home in the
apartment. I started looking for solutions after a few weeks
of procrastination. But before talking about the solution,
we need the last piece of the puzzle in what tools I had at hand.

## Sweet, sweet Raspberry Pi

In a previous apartment, I had a TV in my bedroom, on which I
wanted to watch video content from my computer. This was before
the time of Chromecasts - which have now solved all my
media-related needs - and so, I bought myself a Raspberry Pi
to act as a Media Center. Then times changed and we decided
that we were going to move out of the apartment we lived in to
instead move down to Stockholm. As we had no idea on where we
would live in Stockholm, we deciced not to bring the TV, and
instead assuming that our computer screens would serve as good
enough substitutes until further notice. This meant that I had
a spare Raspberry Pi that served no particular purpose.

After a while I encountered the DNS-based ad-blocking software
called Raspberry Pi, and decided to install it as me and my
partner were both encountering unwanted ads when using apps
on our mobile devices. I also decided to install a few other
things just for fun, such as [`gogs`](https://gogs.io)
(a github-style service that you can self-host),
a few toy projects and so on. But then, one day, I discovered
what I think is the true purpose of my
Raspberry Pi: hosting `home-assistant`!

`home-assistant` is a home automation platform that has
some neat integrations to a lot of IoT products, and using these
integrations then orchestrating these IoT products.
You can find it [here](https://www.home-assistant.io).

## Voluntarily accepting tracking for the greater good

While `home-assistant` allows you to perform actions based
on pretty much anything, for me the most interesting category
of action triggers are based on tracking the inhabitants of
the given home: I achieved this using a concept called
Device Tracking. I opted for the strategy that required the
least amount of action from my partner, who I figured would be
a fairly passive consumer of the home automation services I wanted
to integrate. This meant no possibility of installing apps on
every device involved (which probably would have been bothersome
in many other ways), and instead using the existing network
infrastructure: I opted to use device tracking based on `nmap`.

`nmap`-based tracking (see
[here](https://www.home-assistant.io/components/nmap_tracker/)
for more information) lets us detect which devices are on the
network. If we filter out based on which devices we actually
care about (my phone and my partners' phone), we can perform
actions when either one of us arrives at home, or both of us
have left the home.

## Doing it all automatically

Now we have the entire orchestra in place:

1. A Raspberry Pi to host whatever runs on Linux, essentially
2. IKEA Trådfri lights, which can be interacted with
   programmatically
3. Device tracking through `nmap` which can determine whether
   or not someone is at home
4. A robot vacuum which, as it turns out, can be interacted
   with programmatically

I then decided that the following goals should be automated:

1. The lights should turn off when no-one is home, no matter
   what time of day it is
2. The robot vacuum should turn itself on at the moment when
   everyone has left the home, given that it's during
   acceptable hours to run a vacuum cleaner (daytime)
3. The lights should, if they are on and someone is at home,
   dim down to the minimum amount of light at 21.00, as to
   improve sleep hygiene
4. The lights should, if not during nighttime, turn themselves
   on when someone comes home, as to not require the person
   that comes home to turn them on manually

The last rule is turned off during summertime as it is
essentially always light outside during the Swedish summer.
Hence, we attempt to maximize the amount of natural light
we get.

This has worked really well for us! Less time is now spent
on doing the things we never really wanted to do anyways,
and we can instead spend our time doing the things that
we actually want to do. Which lately has been binge-watching
copious amounts of TV. Figures.

## There's always room for improvement

There are still a few things that I would like to automate, but
have unfortunately not been able to. These are, in order of
importance:

### Turn off the TV when no-one is home

My goal for my morning routine is to get to enjoy it for as long
as possible before having to leave for work, and when leaving
for work, to be able to just run out the door and let everything
sort itself out while I'm away. While turning off the lights
and starting the vacuum cleaner does go a long way towards
that goal, there's still the issue of the TV. I enjoy watching
various things on it in the morning through our Chromecast,
which I know has the ability to turn the TV off through CEC (try
this at home: tell your Google Assistant `turn off the TV`).
I have yet to find a way to trigger this functionality without
using Google Assistant, however. If anyone learns of a way to
do this, please contact me on my social media channels or email.

### Include the window blinds in the light routine

Our apartment is located on the first floor. I've mostly found
this to be fine, and have not been too bothered by the fact that
people can see into our apartment through the windows while
passing by. But a home is a place where you often want privacy,
and so I find myself managing the window blinds manually
frequently.

I'm also keen on maximizing the amount of natural light we get
in our apartment - the windows on our apartment are located in
a north-east direction and hence there is a fairly limited
amount of natural light coming in on any given day. But when
we do get natural light, particularly in the summer, I would
like us to be able to use that instead of using artifical light,
if only to not waste electricity.

I know that IKEA has introduced some automatic window blinds
in the Trådfri series, and I may just check that out if
I feel the need to fill out some spare time.

### Plant management

To be honest, I'm a bit unqualified when it comes to managing
my plants. I would like to get better at managing them, and
when I do that I would probably want to manage the amount
of water and light that these plants get automatically.

That's far away on the horizon though, I think.

### Other things I don't even know about

I've been considering the Nest-series of products which could
be interesting. Camera at the door? Fire alarm? We shall see
in the future. The potential is limitless when you have
`home-assistant` at your side.
