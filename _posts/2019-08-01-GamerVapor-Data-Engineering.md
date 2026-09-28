---
layout: post
title: "GamerVapor, Part 2: Crawling a Social Network (the Data Engineering)"
date: 2019-08-01 00:02:00 -0400
series: gamervapor
series_title: "GamerVapor, my Insight Data Science project (2019)"
---

{% include series.html %}

*Written in September 2026, looking back; dated to when the project was built.*


[Part 1]({% post_url 2019-08-01-GamerVapor-Predicting-Churn-on-Steam %}) introduced GamerVapor, a 2019 tool that predicted which users would leave the Steam community. Before any modeling could happen, I needed a dataset, and there wasn't one to download. This post covers how I built it.

## The source: the Steam Web API

Valve publishes a public web API (part of Steamworks) that returns information about any user whose profile is public. Three kinds of data mattered:

| Data | What it contains |
|---|---|
| **User profile** | When the account was created, profile and privacy settings, location, group (clan) membership, avatar, name, persona state, and **last login** |
| **Friendships** | Pairs of users, with the date each friendship started |
| **Games** | Games owned, and lifetime playtime for each |

The last-login time gave me the label: anyone who hadn't logged in for more than three months counted as churned. The friend list gave me something just as valuable: a way to find more users.

## Network iteration: letting the friend graph do the sampling

![Network iteration: start from one user, then crawl their friends, then their friends' friends](/assets/images/gamervapor/network_iteration.png)

There's no API call for "give me a random sample of Steam users", but every user's friend list points to more users. So the collection was a crawl through the friend graph:

1. Start from a seed user.
2. Fetch their profile, games and friend list.
3. Add each friend to the queue, then repeat for them, and for their friends.

Iterating outward like this, I collected and studied about **200,000 users**.

It's worth being honest about what this kind of sampling does. A crawl along friendships over-represents connected users and under-represents loners, because you can only reach someone through a friend. For a project whose main finding turned out to be about friendships, that's a bias to keep in mind. It's also why the crawl order carried information of its own: for each user I could look "up" the tree (the friends through whom I'd found them) and "down" it (the friends I discovered through them), and some of those counts became features.

## Public and private profiles

Steam users choose what their profile shows. Some profiles are public, with games, playtime and friends all visible. Others are private, so the API returns little more than a name and a login time.

![Churn probability for users with rich public profiles](/assets/images/gamervapor/split_high_info.png)
![Churn probability for users with little public information](/assets/images/gamervapor/split_low_info.png)
*An early version of the model split users into high-information (top) and low-information (bottom) groups, with a separate classifier for each.*

Missing data here isn't random: choosing to hide your profile says something about how you use the platform. My first model version treated the two groups separately, with two classifiers. The final version folded profile settings in as features, so a single model could learn from them directly.

## Storing it

The crawl results went into **PostgreSQL**, and everything downstream (cleaning, joins, feature building) was done in **pandas**. The data is naturally relational: users, friendships between pairs of users, and user–game ownership. A friends-of-friends statistic, such as the average number of friends your friends have, is just a couple of joins.

## From raw data to features

The raw tables became **17 features** per user, in four groups:

- **Individual:** for example, whether the user has a custom avatar, and when the account was created
- **Community:** for example, number of friends, and when the newest and oldest friendships started
- **Game:** for example, total lifetime playtime, and games owned but never played
- **Hybrid:** combining a user with their network, for example whether they share a favorite game with their friends, or how many games their friends own

Part 3 covers which of those mattered, and how much.

## Looking back from 2026

The core pattern still holds up: find a source, crawl it respectfully, store it relationally, and turn it into features. A few things I'd do differently today:

- **Snapshot over time.** One crawl gives you one moment. Repeating it weekly would give the model real before-and-after data, instead of inferring change from timestamps.
- **Think harder about the sample.** A crawl along friendships gives you a very particular slice of the platform, and I'd measure that bias explicitly.
- **Separate label time from feature time.** Build features only from data before the churn window starts, so the model can't peek at the outcome. More on why that matters in [Part 3]({% post_url 2019-08-01-GamerVapor-Data-Science %}).
