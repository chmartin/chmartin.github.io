---
layout: post
title: "Asking Your Data Questions in Plain English: A Hackathon Chatbot That Wrote SQL"
date: 2026-09-28 00:04:00 -0400
---

In 2023, while I was a Staff Data Scientist at LiveRamp, I built a Slack chatbot that turned plain-English questions into SQL. This post looks back at it three years later, so I'll try to describe it the way it looked at the time, not with today's tools in mind. It went from idea to working proof of concept in three days, and it took **first place in the company's hackathon**. Here's the problem it solved and how it worked.

## The problem: data QA needed an analyst

My team measured advertising effectiveness: did the people who saw an ad convert more often than similar people who didn't? Those measurements are only as good as the data going into them. As we used to say, garbage in, garbage out.

Checking that data was a real time sink. Every "does this look right?" meant writing SQL against large tables, so it always needed a technical analyst. Non-technical stakeholders with questions couldn't answer them themselves, and the analysts who could were busy.

So the question for the hackathon was: **what if anyone could ask the data a question in plain English, right where they already work?**

## The state of the art in 2023

It's easy to forget how new all of this was. ChatGPT had launched at the end of 2022, and OpenAI's API for its chat models only arrived in early 2023. Most companies were still working out whether they were allowed to use it at all.

A few things we take for granted now didn't exist yet, or were brand new:

- **Small context windows.** The ChatGPT model behind the API could take about 4,000 tokens in total. Every table definition you added to the prompt came out of the same small budget as the question and the answer, so what to include was a real design decision.
- **No agents to lean on.** The idea of a model that runs a query, looks at the result, notices a mistake and tries again on its own wasn't something you could build reliably in a hackathon. You got one response per request, and you designed around that.
- **Text-to-SQL was a research problem.** Models could write plausible SQL, but they would happily invent tables and columns. Nobody trusted a query just because a model wrote it.
- **Data privacy was the first question.** Sending company data to an outside AI service was a non-starter in a lot of places.

The design below comes directly from those constraints.

## The design

![Architecture of the natural-language SQL Slack bot](/assets/images/nl-sql-chatbot.png)
*The architecture, from my hackathon slides.*

Everything happens in a Slack bot, so there's nothing new to install or learn:

1. **Ask.** A user types a question such as *"What does my data look like?"*
2. **Build the prompt.** A prompt constructor combines the question with the **table definitions (DDL)**, so the model knows which tables and columns actually exist. Only the schema was sent to the model, never the data itself; the query ran on our side. Choosing which table definitions to include was part of the job, given how little fit in the prompt.
3. **Generate candidates.** The prompt goes to a large language model. For the hackathon that was ChatGPT's API. The field was changing month to month, so I didn't want to be tied to one provider: the model sat behind a small natural-language API layer, and it could be swapped for an internal model or another service.
4. **Let a human choose.** The bot doesn't run the first answer it gets. It shows the user **three possible SQL queries**, and the user picks one (*"Choice 2"*). In 2023 this was the honest answer to "how do you know the SQL is right?": you asked a person. Three candidates made it more likely one was correct, and the person choosing acted as the check. The plan was to take humans out of the loop once we trusted the system.
5. **Run it and show it.** The chosen query runs against the database. The bot replies with a preview in Slack, a plot where one makes sense, and a link to the full results.
6. **Keep the pairs.** Every prompt and response pair was saved. Fine-tuning your own model was just becoming practical, so these pairs were meant to become training data for a model tuned to our own schema and questions.

## Why it worked as a hackathon project

The project stood out for its company impact and its design, and a few choices mattered most:

- **It met people where they were.** A Slack app meant zero adoption cost, with no new tool or login.
- **The schema was the context.** Giving the model the real table definitions kept its queries grounded in columns that exist.
- **Humans stayed in control.** Offering three candidates and letting a person choose made mistakes visible and cheap, and it quietly collected labeled data at the same time.
- **Nothing was locked in.** Keeping the model behind an API layer meant the provider could change without touching the bot.

It was a working proof of concept in three days, and it won first place.

## What came next

The plan for the next version was to move to Google's Vertex AI, which had just started offering generative models for enterprise use, and an internally tuned model, trained on the prompt and response pairs the first version collected, so that over time the human-choice step could be dropped for routine questions.

## Looking back

A lot of what this project had to work around is now built in: models with far larger context windows, tool calling, and agents that can run a query, check the result and fix their own mistakes. But the core ideas have held up. Give the model the right context, never trust a single answer without a check, keep a human in control until the system earns trust, and save what works so the system can improve. In 2023 all of that had to be designed by hand, in three days.
