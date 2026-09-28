---
layout: post
title: "Asking Your Data Questions in Plain English: A Hackathon Chatbot That Wrote SQL"
date: 2026-09-28 00:04:00 -0400
---

In 2023, while I was a Staff Data Scientist at LiveRamp, I built a Slack chatbot that turned plain-English questions into SQL. It went from idea to working proof of concept in three days, and it took **first place in the company's hackathon**. Here's the problem it solved and how it worked.

## The problem: data QA needed an analyst

My team measured advertising effectiveness: did the people who saw an ad convert more often than similar people who didn't? Those measurements are only as good as the data going into them. As we used to say, garbage in, garbage out.

Checking that data was a real time sink. Every "does this look right?" meant writing SQL against large tables, so it always needed a technical analyst. Non-technical stakeholders with questions couldn't answer them themselves, and the analysts who could were busy.

So the question for the hackathon was: **what if anyone could ask the data a question in plain English, right where they already work?**

## The design

![Architecture of the natural-language SQL Slack bot](/assets/images/nl-sql-chatbot.png)
*The architecture, from my hackathon slides.*

Everything happens in a Slack bot, so there's nothing new to install or learn:

1. **Ask.** A user types a question such as *"What does my data look like?"*
2. **Build the prompt.** A prompt constructor combines the question with the **table definitions (DDL)**, so the model knows which tables and columns actually exist. Without them, a model will confidently write SQL against columns that aren't there.
3. **Generate candidates.** The prompt goes to a large language model. For the hackathon that was ChatGPT, but the design wasn't tied to it: the model sat behind a small natural-language API layer, so it could be swapped for an internal model or another provider.
4. **Let a human choose.** The bot doesn't run the first answer it gets. It shows the user **three possible SQL queries**, and the user picks one (*"Choice 2"*). This response-evaluation step was a deliberate human in the loop. The plan was to take humans out of the loop once we trusted the system.
5. **Run it and show it.** The chosen query runs against the database. The bot replies with a preview in Slack, a plot where one makes sense, and a link to the full results.
6. **Keep the pairs.** Every prompt and response pair was saved. Those pairs were meant to become training data for fine-tuning a model on our own schema and questions.

## Why it worked as a hackathon project

The project stood out for its company impact and its design, and a few choices mattered most:

- **It met people where they were.** A Slack app meant zero adoption cost, with no new tool or login.
- **The schema was the context.** Giving the model the real table definitions kept its queries grounded in columns that exist.
- **Humans stayed in control.** Offering three candidates and letting a person choose made mistakes visible and cheap, and it quietly collected labeled data at the same time.
- **Nothing was locked in.** Keeping the model behind an API layer meant the provider could change without touching the bot.

It was a working proof of concept in three days, and it won first place.

## What came next

The plan for the next version was to move to Google's Vertex AI and an internally tuned model, trained on the prompt and response pairs the first version collected, so that over time the human-choice step could be dropped for routine questions.

## What I took away

The part I'm proudest of is also the least flashy: the loop. Give the model the right context (the schema), let it propose options, have something reliable check them (at first, a person), and keep a record of what worked. That's what turned a demo into something that could get better with use.
