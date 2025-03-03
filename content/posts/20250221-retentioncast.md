+++
title = "RetentionCast"
author = ["Eoin H"]
publishDate = 2025-02-21T00:00:00+00:00
tags = ["post", "project"]
draft = false
+++

# RetentionCast: Building Real-Time Churn Prediction with Change Data Capture

## TL;DR

This is a quick write-up of RetentionCast, a system I built to predict customer churn in real-time using Change Data Capture. The system processes live transaction data through Kafka and Debezium to calculate churn probability and Customer Lifetime Value. Check out the [GitHub repo](https://github.com/eoinhurrell/RetentionCast)

## The Churn Problem

If you've worked in e-commerce or SaaS, you know customer churn is the silent killer of growth. Traditional approaches to churn prediction often involve batch processing - running models overnight on yesterday's data. But by the time you've identified that a customer is at risk, they might already be gone.

Some companies model churn as "if we haven't seen them in X days they've churned". This is a poor model because it neglects the customer's actual usage of the system. If a user who visits every day hasn't been seen in three days they could be gone. This has always bugged me. The data to identify churn risk often exists in your system the moment a customer takes (or doesn't take) certain actions. What if you could catch these signals the moment they happen? A better way to look at churn is to use more complex models like the 'Buy Til You Die' models, that can project likelihood of churn for each user forward in time (i.e. if we haven't seen this person in three days they would have a 60% chance of having churned).

I was a big fan of [lifetimes](https://github.com/CamDavidsonPilon/lifetimes), a Python package for analyzing purchase behavior and predicting customer lifetime value, before it was archived. Its spiritual successor [PyMC-Marketing](https://github.com/pymc-labs/pymc-marketing) has incorporated much of its functionality, but I wanted to go deeper on the underlying algorithms.

RetentionCast gave me an excuse to reimplement some of these models from scratch, but with a twist - making them work in a streaming context rather than batch.

## Real-Time with Change Data Capture

Change Data Capture (CDC) is one of those powerful patterns that isn't discussed enough. The premise is simple: instead of querying your database for changes, you tap directly into the transaction log and stream out changes as they happen.

It's getting much easier to build these types of streaming architectures. In previous work when setting up real-time analytics pipelines, it was both exciting and frustrating because the tooling wasn't there and you had to build basically every piece yourself. Now with tools like Debezium and Kafka, the heavy lifting is handled for you.

Here's how RetentionCast's architecture works:

1. As transactions happen in the e-commerce system, they're written to Postgres
2. Debezium captures these changes directly from Postgres's transaction logs
3. These changes are streamed to Kafka topics
4. A consumer service processes each event to update an analytics table with key metrics

## The Analytics Table

The heart of the system is surprisingly simple - a single table with these fields:

- `first_purchase_date`: When the customer first converted
- `last_purchase_date`: When the customer made their most recent purchase
- `frequency`: Number of repeat purchases
- `total_order_value`
- `avg_order_value`

From these basic metrics, we can derive powerful insights about customer behavior patterns and churn likelihood. For this project I store `retention_campaign_target_date`, the date at which the user's likelihood to still be alive falls below 70%, i.e. a good time to add them to a win-back campaign to retain them. The trick is keeping this table updated in real-time as transactions flow through the system.

## Why This Approach Works

Churn modelling is usually performed in batch, i.e. a batch operations looks at all users and predicts "who is at risk now", using this to select users to target. RetentionCast is designed to calculate when a user is likely to churn as they are checking out, using survival analysis to have a more nuanced take on churn.

RetentionCast strikes a balance by:

1. Processing each transaction as it happens
2. Updating key customer metrics immediately
3. Making those metrics available for real-time prediction

The result is a system that can tell you, at any moment, which customers are showing early warning signs of churn - when there's still time to intervene.

## The Trade-offs of Real-Time

It's worth noting the downsides of this approach. As Dan McKinley famously wrote, ["Whom the Gods Would Destroy, They First Give Real-time Analytics"](https://mcfunley.com/whom-the-gods-would-destroy-they-first-give-real-time-analytics). Real-time systems are:

- More complex to operate than batch systems
- Potentially more difficult to debug and fix in production
- A trade-off that only makes sense at scale or for critical business functions

## What's Next?

RetentionCast is still a work in progress. I've got plans to:

- Add more sophisticated churn prediction models, i.e. different churn models for different segments of the user base.
- Build visualization dashboards for monitoring
- Benchmark performance at different scales

If you're interested in exploring this further, check out the [GitHub repo](https://github.com/eoinhurrell/retentioncast). I'd love feedback, especially from folks who've tackled similar problems in production environments.
