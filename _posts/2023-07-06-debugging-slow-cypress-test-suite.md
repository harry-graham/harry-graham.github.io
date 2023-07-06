---
layout: post
title: "Debugging Slow End-to-End Tests 🐢"
date: "2023-07-06"
---

This is a story of how I managed to reduce a Cypress test suite's total runtime by 83%.

## Introduction

### When slow-running tests become too painful to ignore ⛔

At work, I had to add a new endpoint to a TypeScript API. This was a nice and simple piece of work, nothing complex.

However, when I ran the end-to-end tests locally, they took 52 seconds to run!

Now, you might be thinking, _"Wow, 52 seconds, that's so quick!"_. Yes, it is quick compared to some test suites, but:

- These tests were each supposed to be taking ~100ms or so to run, so the full test suite should've been finishing within 10 seconds maximum.
- This doesn't actually reflect the true total time to run, it only reflects the time recorded by Cypress (i.e. it doesn't reflect the time Cypress takes between tests), so it's actually more painful than it sounds.
- This was a very new codebase, and whilst I'd been off work for a few weeks, the test suite's total runtime had become what seemed like 10 times slower.

Also, it's worth noting some things about test suites generally:

- It is essential to keep total test suite runtime as low as possible, especially when running the tests locally, to avoid interrupting our development flow, and therefore to keep our engineers happy.
- In the long run, faster test suites are less painful, more fun to work with, and generally add so much value.

So, I really felt this was something that needed fixing.

I wasn't surprised by the slow tests: a month or so before this, after returning to work following leave, I had noticed them and I just hadn't gotten around to investigating the root cause yet. Since it was now affecting me, I couldn't resist trying to address the issue.

Fortunately, I had been having a really productive day at work, and felt I had time available to comfortably spend a few hours looking into this.

### Context: The App

The app was an API that was:

- Written in TypeScript
- Using Koa as a web framework
- Using Cypress for end-to-end tests
- Running the tests in a docker container

## Chapter 1: The Investigation Begins

### Where to look first? 🤔

To start with, I ran the full end-to-end test suite locally.

Immediately, in the Cypress summary, it was clear that most of the test files were ok, and that there were only a few test files that were taking excessively long compared to the rest.

[Screenshot of Cypress summary, highlighting the slow running tests]

So, starting with the first of the slow test files, I zoomed in on the Cypress summary/breakdown for that test file, and it was clear that only some of the tests were slow.

[Screenshot of Cypress summary for that test file, highlighting the slow-running tests]

Next, I opened up the test file and read through the first offending test. Whilst there were several things I would've changed about the test itself, there wasn't anything obvious that would take so long to run.

I decided to dive deeper. Since the tests were being run in a docker container, I wasn't sure of how to add breakpoints to the test to be able to stop partway through the code, like I would when using Pry on Ruby and Rails projects. So, I decided to take the basic approach of manually adding console logging lines throughout the test, to hopefully demonstrate where the slowness was.

[Gif of running a Cypress test with console logging]

Hmm.

All of the logging happened very quickly, there wasn't anything slow happening within the test as far as I could tell.

What if the slowness was happening during the test setup/teardown phase? It shouldn't have been, since the app was using the same setup/teardown logic for all tests, even the fast ones. I tested that using console logging, and surely enough, all of the logging happened very quickly again.

I was stumped. Somehow, the tests were recording taking forever to run, but the test logic seemed very fast. What was causing the delay?

### Some key observations

Whilst I sat there flicking through the test code and console outputs, hoping to magically find the root cause, I suddenly noticed something: the duration of each slow test seemed to be multiples of 5 seconds, e.g. ~5s or ~10s.

An idea occurred to me. I quickly checked, and confirmed that each slow test called certain POST endpoints, and in fact, each call to those endpoints seemed to add another 5s to the test. That is:

- Tests that made 1 endpoint call took ~5s
- Tests that made 2 endpoint calls took ~10s

But what was causing these 5 second delays?

At this point, it became clear that I needed to start manually testing those endpoints, and that perhaps this wasn't just a test suite issue.

## Chapter 3: Debugging the endpoint 💻

I started up the app locally and manually tested the first affected endpoint using Postman.

The request seemed to complete reasonably quickly, but the total duration kept ticking on until reaching ~5s.

I decided to dive deeper by adding lots of breakpoints throughout the code and running a JavaScript Debugging Terminal in VSCode, which I stepped through line-by-line. Nothing seemed to take a long time, but the request still seemed to take around 5s after the response had seemingly returned.

Fortunately, something caught my eye: the response body was `null`.

### Confirming the root cause

I had been part of the team that had written this application over the previous six months, and we had been returning `undefined` as the response body whenever we weren't returning anything. I hadn't seen us return `null` before, but surely that couldn't be adding 5 seconds here, could it?

For completeness, I decided to check it anyway, and lo-and-behold, the response was returned in ~XXms!

[screenshot of faster response time]

At this stage, after doing some exploring and clicking, I realised that Postman actually provides a clear breakdown of the response time into different categories. I'm still not sure what each part represents, but when returning `null` from the endpoint, the response had a 5s download time.

Using VSCode's search functionality, I could see that we were using a `null` response body on the exact endpoints whose tests were running slowly, so this was clearly the root cause.

### Checking for Impact

Before doing anything else, I tested one of these endpoints from a downstream application to check for potential impact, and fortunately, the consumer session didn't seem to be affected, so the impact only seemed to be limited to the test suite and CI pipeline.

## Chapter 4: Koa's Unexpected "Feature" 🔍

I did some Googling, and managed to find a Koa issue discussing how returning a null response body caused Koa to remove some key headers from the response. Checking the Koa code, I could see that this including removing the `Content-Length` header.

I mentioned this to my manager, who confirmed that this likely meant that the receiver (in this case, Cypress) couldn't tell in advance how long the response would be, and so instead defaulted to attempting to download the response until it hit an internal limit of 5s.

## Chapter 5: The Result 📊✅

After changing the culprit endpoints to return a response body of `undefined` instead of `null`, I reran the full test suite, and saw that the total runtime of the test suite was now only ~8s! 🥳🎉✨

[screenshot of final test summary]

In particular, one of the test file's total runtime had dropped from 37s to <1s!

[before-and-after screenshot of the test summary for 37s to 1s]

The overall reduction from ~52s to ~8s was an ~83% reduction, and represented a massive quality-of-life improvement for our engineers.
