---
title: "Bypassing RoyalRoad's piracy nags in RSS Feeds"
date: "2024-11-20"
tags:
  - rss
  - freshrss
categories:
  - selfhosted
---
# Issue

[Royal Road](https://www.royalroad.com/home) likes to annoy pirates. This is (arguably) good.

Royal Road doesn't care if they annoy RSS users. This is **bad**.

Here's a walkthrough of the problem and the fix.

### The Problem:

First, let's look at the full picture of why this is happening.

The Original Website HTML (Simplified)

When you visit the Royal Road chapter in your browser, the full page's HTML looks something like this. Your browser loads the <head> section and the <body> section.
```HTML

<html>
<head>
    <style>
        .cjZhYjNmYjZkZmFjZTQ2YTk4OWQwYjRiMjRjZDQyOGRl {
            display: none;
        }
    </style>
</head>

...

<body>
    <div class="chapter-content">

        <p class="cnMxYzY0ZjllNmVj...">
            <span style="font-weight: 400">Nathan got the message...</span>
        </p>

        <p class="cnNiYTMwZmE4YjE2...">&nbsp;</p>

        <p class="cnNiOWQ0MDU1MDA2...">
            <span style="font-weight: 400">Sarya waved her hand...</span>
        </p>

        <p class="cnM0NjAwNWU4Y2Vl...">&nbsp;</p>

        <span class="cjZhYjNmYjZkZmFjZTQ2YTk4OWQwYjRiMjRjZDQyOGRl">
            <br>The narrative has been stolen; if detected on Amazon, report...<br>
        </span>

    </div>

    </body>
 </html>
```


On the live website, your browser reads the `<style>` tag in the `<head>` and knows to hide the spam `<span>`. You never see it.

### What FreshRSS Sees (The Problem)

I've told FreshRSS to only grab the content from `.chapter-content` which is the actual content of a post. So, FreshRSS requests the page and then scrapes only this part:

```html

<p class="cnMxYzY0ZjllNmVj...">
    <span style="font-weight: 400">Nathan got the message...</span>
</p>

<p class="cnNiYTMwZmE4YjE2...">&nbsp;</p>

<p class="cnNiOWQ0MDU1MDA2...">
    <span style="font-weight: 400">Sarya waved her hand...</span>
</p>

<p class="cnM0NjAwNWU4Y2Vl...">&nbsp;</p>

<span class="cjZhYjNmYjZkZmFjZTQ2YTk4OWQwYjRiMjRjZDQyOGRl">
    <br>The narrative has been stolen; if detected on Amazon, report...<br>
</span>

```

Since FreshRSS never saw the `<head>` or the `<style>` tag, it has no idea it's supposed to hide the spam `<span>`. It just displays all the text it found, resulting in this output in your feed reader:

    Nathan got the message...

    Sarya waved her hand...

    The narrative has been stolen; if detected on Amazon, report...

This is the core of the issue: the content is hidden by a CSS rule that FreshRSS isn't loading, and the class names are random, so you can't just block the class.

### The Fix: CSS Selectors

You need to tell FreshRSS how to remove the unwanted elements based on their structure, not their random class names.

Go to: **Advanced** -> **CSS selector of the elements to remove**.
Paste this in the box:

```css
.chapter-content > span
```


This selector targets any `<span>` element that is a direct child (using `>`) of `.chapter-content`.

The spam text `<span class="cjZhY...">...</span>` matches this rule.

The actual story text `<span style="font-weight: 400">...</span>` is safe because it's a "grandchild" (it's inside a `<p>` tag), not a direct child.
