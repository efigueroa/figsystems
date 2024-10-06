+++
title = 'RSS - Still Alive'
date = 2024-10-05T21:00:05-07:00
tags = ['rss','selfhosted']
categories = ['selfhosted']
+++

# RSS - Still Very Useful

I like having a centralized curated list of content. I'd rather go to a single page to catch up on new content instead of visiting or remembering to visit a bunch of different sites. I also don't like having to deal with cookies and sites tracking my every move. 

Intro: [FreshRSS](https://freshrss.org/index.html) and [RSSHub](https://docs.rsshub.app/) with special guest [RSSHub-Radar](https://github.com/DIYgod/RSSHub-Radar)

I used to only use RSS for blogs and other text based content but with the above tools I can RSS-ify most anything. 


## The Flow of Content ##

**FreshRSS** is the rss aggregator and can be used as the reader either on desktop or as a PWA on mobile. I host my own at [feeds.fig.systems](https://feeds.fig.systems). It's got a few slick themes and has options to [scrape webpages with x-paths](https://danq.me/2022/09/27/freshrss-xpath/). You provide a URL and use elements to select what you'd like to create your feed from, that's a lot of work per feed. 

That was my first go at RSS-ifying everything until I learned about **RSSHub** which does the same thing but completely for you.  I have my own hosted RSSHub.fig.systems. They provide pre-made "routes" which make turning many common content sources into feeds.

For example I could add the feed ```rsshub.fig.systems/youtube/user/linustechtips/``` to feeds.fig.systems and I'd get a new entry every time that channel had a new feed. 

Having to look up the routes can be annoying. Enter **RSSHub-Radar**, a nice browser extension that can automatically detect and provide the route for a given page you'd like to rss-ify. The extension can also be configured to format the url for whatever rss aggregator you use, FreshRSS or otherwise. 

It's worth going over [RSSHub's routes](https://docs.rsshub.app/routes/popular) to get an idea of what can be turned into an rss feed. 
