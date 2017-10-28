---
layout: post
title: Indie Developer Guide to Production, Intro
tags: [Startup]
description: Best-practices, lessons-learned and little tricks that I've collected in the process of building and releasing several web services from scratch to production.
---

In the following posts I will share best-practices, lessons-learned, IDE configurations and little tricks that I've collected in the process of building and releasing several web services from scratch to production.

[Hacker News Books](http://hackernewsbooks.com/) \(HNB\) is an book discovery service, running autonomously since August 2016,  analyzing comments on Hacker News \(HN\) to extract links to books on Amazon, Safari Books, and O'Reilly, aggregating them, and then ranking them based on votes, karma, and other criteria. It landed on place first on [Hacker News](https://news.ycombinator.com/item?id=12365693) during launch, got featured on [Indie Hackers](https://www.indiehackers.com/businesses/hacker-news-books), made more than [$1000 dollars in 5 days after launch](http://hackernewsbooks.com/blog/making-1000-dollars-in-5-days-with-amazon-associates) and has collected more than 1500 happy tech-savvy readers on its newsletter in only a couple of months.

[HackerPixels](http://hackerpixels.com/) is a video newsfeed, extracting Youtube or Vimeo links from comments on HN and generating a newsfeed, with video preview and search. It was praised on [Product Hunt](https://www.producthunt.com/posts/hacker-pixels) and unearths plenty of tech, geeky or just funny videos that you probably have not heard of.

Building a product from scratch may seem intimidating at first but it is a very achievable and rewarding experience. Once you are all set up with the help of this guide, then you will be able to iterate and try out new things really fast. Hacker News Books was not the first thing I tried, but without the previously failed attempts it would have failed too. So stop procrastinating and start building one step at a time.

If you follow through the posts you will have a very reasonable IDE setup with PyCharm, that keeps your development environment sane and allows you to focus on the actual business logic. Your deployment setup will be the same for local and production releases, which allows frictionless upgrades without having to write your own code for rolling back in case of unexpected issues. The production setup will configure your services in a redundant and safe manner, exposing only what is needed. Happy indie hacking!

