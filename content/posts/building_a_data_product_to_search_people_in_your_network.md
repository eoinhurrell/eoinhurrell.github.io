+++
title = "Building a Data Product to Search People in Your Network"
author = ["Eoin H"]
date = 2017-10-06T00:00:00+01:00
tags = ["products", "design", "engineering"]
draft = false
+++

_The below was originally written for and [published on](https://medium.com/cohort-analysis/building-a-data-product-to-search-people-in-your-network-5443ab992e90) the Cohort company blog,
Oct 6, 2017._

Dr. Eoin Hurrell leads Data at Cohort. Here, he reflects on what’s involved in
designing a system to ingest and analyze massive volumes of public data, to make
predictions on relationships, and things people are knowledgable on.

Cohort helps you find the people you need, through the people you already know
and trust. It sounds simple, but the experience of existing social search is
much closer to “I am looking to follow/connect with a person on this site”.

This makes it hard to know who you could ask for help from, or even who your
friends actually know and can introduce you to on a social network’s friends
list. This post will talk about the data-driven approach we have taken, to make
searching your network in a real-world context possible.


## Problem Outline {#problem-outline}

The purpose of these searches is to highlight not just friends but
friends-of-friends — people your friends could introduce you to to help you
solve a problem. This has different implications for relevance than other search
engines, as ideally it should surface people who your friends know well, who
know something about the thing you are searching for, and possibly aren’t
obvious to you within your own network.

This indicates a subjective strength of relationship variable not present in
other networks, i.e. is this relationship strong enough for one person to ask a
favour of the other? The results should also highlight things about your network
that you don’t know, teaching you about your wider context.

Cohort’s unique search also presents challenges for setting up an index to
search, as each user is searching their own unique view of the world — their
friend-of-a-friend network. Our data strategy, the motivating principles which
define how inputs and generated data are handled, is key to a successful search
experience.


## Solution Overview {#solution-overview}

As a quick overview, we have constructed a data ingestion pipeline using Kafka
that crawls social graphs, both actively (when a user signs up) and passively.
The resulting information about people is stored in a single source of truth (a
single datastore that holds all the data we’ve seen already, to ensure
consistency by avoiding duplication of people) and rendered as documents in our
search index. (Watch out for a follow up post, exploring this in more detail.)

As we crawl social graphs we make a number of predictions using machine learning
models to improve the search experience by adding information to each person’s
document in the index. This information enriches these documents and allows them
to be considered relevant for more search results. Semantic understanding of
text with interest-focused natural language processing models is used to help
understand the areas of interest a social profile describes itself with or talks
about in its feed. (Again this will be explored in a follow up post.)

We also have a model to predict the likelihood that a relationship between two
people, each modelled using many features from their social profiles, is strong
enough that one would contact the other to request a favour, which helps
personalize search results for a given user. This prediction feeds into the
subjective rather than objective search Cohort provides.


## Data Modeling {#data-modeling}

Simple models are preferable, as they can be more easily interrogated and
tweaked. Heuristics are preferable to complex models in the early phases of a
project. The most sophisticated algorithms are a waste of time without the right
data to learn from for the task. Amazon [have written](https://arxiv.org/pdf/1705.02245.pdf) about this and talk about
the concept of data readiness, the idea that data should be evaluated in
relation to the problem it is being used to solve, and fully understanding the
usefulness of data available makes understanding the nature of the problem much
easier.

One of the trade-offs we made early on in forming our data strategy was to focus
on a single source of social data for ingestion, where multiple sources might
seem preferable. Multiple sources however bring a potential explosion of
duplicate data and possibly even models (as we must try and predict relationship
strength among users of different networks with very different features) as well
as operational issues. Disparate data and disparate behaviours are much harder
to learn from (both in researching user behaviour and training models), because
each source brings only part of a picture. It is better to work well on one
source than poorly on many possible sources.

Ultimately Cohort becomes more powerful the more it is interacted with, rather
than the more data from other sources (with other purposes).

We were careful in our modelling to choose either heuristics that closely
followed what little data we had when we began working, or model an analogous
prediction that was easier to get training data for. We have closed the loop by
relearning from the data we have gathered from our users since we launched, to
good effect.


## User Experience {#user-experience}

Users of the app have the opportunity to, through their actions, alter areas of
interest and relationships predicted for them. Rather than having a single
evaluation on a test set, a machine learning model in a data product is a living
thing, where models may need to be retrained or predictions inspected.

Structured search, with suggestions, using Cohort.

At query time a user’s experience of the system should look something like this:

1.  A person inputs the area of interest or free text query they want to find
    people relevant for
2.  As the user queries the system an auto-complete displays the areas of
    interest we have observed in the documents we have indexed, teaching them
    about the system’s understanding
3.  A smart multi-match query is formed from the search terms the user enters,
    to figure out what the user wants, matching our machine-learning driven
    components, the raw profile information and other derived content
4.  The scoring function for this query takes into account the text searched
    for, whether it was an predicted area or free text, and the social affinity
    of the person within the user’s network, returning a list of relevant people
5.  We also help refining this search using market-basket analysis to offer
    further known areas of interest that could be added to their search,
    allowing them to refine their search transparently
6.  The user finds someone relevant that a friend can introduce them to and
    starts a conversation to get an introduction.


## Data Engineering & Infrastructure {#data-engineering-and-infrastructure}

A huge factor in the successful deployment of a data product is investing in the
tooling and infrastructure to record data about your production system so that
improvements can be measured quantitatively. As models and query-parsing
algorithms are pushed to production it is important to measure their impact as
clearly as possible. The production environment is the experimental environment,
you will get the best data on how well any model or algorithm works there, be
sure you can record it.

Logs of items returned and more importantly items the user chose to engage with
for a given query, called click logs, are the data that is usually collected
from a search engine, as precision, recall, nDCG, f1 score and other common
search metrics are derived from what was shown and proved to be engaging. This
can help in multiple ways, feeding back into models, informing design, allowing
you to calculate KPIs or to focus on a specific user’s experience of the
product. They have also lead to new modifications to our scoring function to
better improve relevance, lead to removing query expansion for more concise
results.


## Conclusion {#conclusion}

The experience of machine learning models is often vastly under-examined in data
products, but reports and reactions to predictions and recommendations is
valuable qualitative feedback. People will frequently make assumptions about how
algorithms work that may have an impact (positive or negative) on their
experience and be eager to talk about it.

Recommendations generated using collaborative-filtering can be felt and
explained as a deep understanding of the user’s interests, while other reactions
could draw attention to a lack of result-set diversity. This feedback can
influence the implementation of the production models to provide a better
experience.

Most production data science work is ensuring you have the right data in the
right place at the right time, acquiring and cleaning what’s needed to make a
prediction/highlight relevant items. Cohort’s search is a data product designed
to solve a complex real-world problem with the data available and provide a
great experience to the user as a result. Where we are now we have a complete
system that we can add complexity and functionality, like query expansion or
learning to rank models, to as we research and act on feedback loops generated
from usage.
