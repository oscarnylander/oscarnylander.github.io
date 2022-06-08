---
title: "Automating the boring stuff with Github Actions"
layout: post
description: "How I built a Github Action to automatically remove unused resources in Android-repositories"
---

# Automating the boring stuff with Github Actions

{{ page.date | date_to_string }}

Recently, at work, I got inspired after having added Renovate to our android repository,
and decided that I wanted to increase the level of boring stuff-automation for our team.
This is the story on how I used Github Actions to automatically and periodically remove
unused resources in our Android-repository.

## Prior art

Since this was merely a small side-project, I did not quite have the time to implement
a checker for which resources were unused. Luckily, I could leverage the excellent
gradle plugin called `gradle-unused-resources-remover-plugin`. You can find this plugin
at https://github.com/konifar/gradle-unused-resources-remover-plugin, it does basically
all of the heavy lifting.

## How to call a plugin that is not added to your project

Ideally, since this plugin is only really useful when it's being called, we want to
avoid adding it to the project permanently, since that would slow down the configuration
of Gradle. Here's how I tackled this problem:

1. Create a file containing a buildscript-block with the gradle plugin portal repository,
   and a classpath-dependency on the aforementioned gradle plugin. Finally, apply the
   plugin at the end of this file
2. Append an `apply from`-statement to the root build.gradle-file, including our little
   addon-gradle file

This makes the plugin available for execution, without it having already been added to
the repository previously. This little trick is quite useful, and can be re-used to
create more of this category of Github Actions to repositories with Gradle.

## Calling the plugin, cleaning up, and optionally making a PR

After having included the plugin, we can merely call it, and have it update the
workspace. We can then leverage the `create-pull-request`-action to create a PR with
these changes (you can find it at https://github.com/marketplace/actions/create-pull-request).
The action does not create any PR if no changes are detected in the repository.

A sharp eye may notice that, since we're making some modifications to the build scripts,
there will always be some changes to the workspace. This is true - in order for this
not to create unwanted PRs, we will have to restore the build scripts to their original
state before attempting to call the `create-pull-request`-action.

## Scheduling it

Now that we have everything in place, we need to set the action up to be called at some point.

We could theoretically just do this every time we push to the main branch, but that would
potentially caused a larger than desirable volume of PRs. I opted instead for running it on
a schedule, which Github Action supports. We can use the following slightly cryptic
cron-expression:

```yaml
on:
  schedule:
    - cron: "30 9 * * 0-5"
```

To make the action run at 9:30 on Monday-Friday, giving us a pleasant little PR to check
out some time after our morning standup. Nice!

## Try it out yourself

If this sounds like something you'd like to integrate into your repository, you're in luck -
this action is available for use on Github. You can find it here: 
https://github.com/HedvigInsurance/gradle-unused-resources-action/

For example usage, you can check out where we call this action in our repository, here:
https://github.com/HedvigInsurance/android/blob/4e00a79dbdf065a3f3802b8c5be25d8a7813c849/.github/workflows/unused-resources.yml

Should you encounter any issues, we welcome PRs in the action repository.
