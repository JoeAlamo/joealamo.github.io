---
layout: post
title: "Diagnosing Database Locks"
date: 2022-10-06
excerpt_separator:  <!--more-->
tags:
 - databases
 - mysql
 - mariadb
 - troubleshooting
---

![Busy traffic](https://miro.medium.com/max/640/1*hFNyVz7JezKXZEON-aZurg.gif)

Relational databases have been an area that I've felt I didn't really understand what was happening under the hood, how to optimise queries and how to troubleshoot issues.

You may be able to relate with me, you can get a long way with a basic understanding of queries but when things go wrong it can feel daunting to dig into all of the complexity that a RDBMS is handling for you under the hood.

After encountering a couple of different scenarios at work where database locks resulted in impaired system behaviour, I decided to spend some time to dive more deeply into transactions & database locks in MySQL and MariaDB.

I initially wrote up an internal troubleshooting guide to aid the rest of the team at One Utility Bill, but when we decided to launch our own [Engineering Blog](https://engineering.oneutilitybill.co) I turned it into a proper blog post.

Head over to the [OUB Engineering Blog](https://medium.com/@joe.alamo-keilty/e83c05ac7611) to read the post in full!
