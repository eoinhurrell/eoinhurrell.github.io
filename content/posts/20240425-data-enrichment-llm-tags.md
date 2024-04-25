+++
title = "Data Enrichment with LLMs for Obsidian: Better Tags"
author = ["Eoin H"]
publishDate = 2024-04-25T00:00:00+00:00
tags = ["post"]
draft = false
+++

One issue I have using Obsidian is that I feel I don't use enough of its feature set. I have lots of notes, but few are tagged particularly well, so it makes tags less useful overall. I would like to be able to make surprising connections and surface related items, which means imprecise tags are preferable to no tags, so false positives are less of a concern to me, and very low risk.

# LLM for zero-shot classification

I decided to frame this as a zero-shot classification problem - "Given this text, which of these tags would you classify it as?", with no other training. LLMs have [shown good results](https://www.analyticsvidhya.com/blog/2023/09/power-of-llms-zero-shot-and-few-shot-prompting/) in this task.

## Implementation

I implemented this quite simply using [Ollama](https://ollama.ai/) to serve the LLM and [Langchain](https://www.langchain.com/) for the actual app.

Langchain's Obsidian vault iterator was helpful in processing notes and giving access to tag metadata. For the initial tests I generated tags for notes I already had tags for, so I could see if there was a reasonable overlap between my tags and the suggestions. I then looked at generating tags only for notes that had none.

I also used Langchain's structured output parsing to store the result as yaml.

## Prompting

The prompt was straightforward, I had to do some tuning to stop the model providing commentary either after the tags or for each tag. It pays to be explicit. The possible tags list came from the most frequent tags in my vault, lightly curated. I populated the {text} marker with the full text of the note.

```
TASK:
You are a helpful YAML-writing assistant organizing notes.
You do not write English, only YAML.
Your task is to choose a list of tags from the POSSIBLE TAGS list to represent TEXT TO TAG
Choose a sensible number of these tags (up to six).
Good tags are tags from the given list (POSSIBLE TAGS), that are suitable to apply to a text because they are relevant to it.
If nothing seems relevant reply with an empty list.
Choose the tags, then format them as YAML in your reply
Give valid YAML output of a list of tags as your response, nothing more.
Do not assume a context that is not explicit, do not give reasons for your choices.
Be concise, only choose the most fitting to use for the text given and do not create tags from the text.
Ensure the tags you choose are from the POSSIBLE TAGS list.
A list of tags starts with "tags:" and is each item on a new line preceeded by a "-", for example:
tags:
  - horror
  - fiction
---

POSSIBLE TAGS:
  - science
  - politics
  - career
  - history
  - workflow
  - writing
  - leadership
  - generative_ai
  - work
  - machine_learning
  - society
  - speculative_fiction
  - reading
  - tech
  - irish
  - economics
  - health
  - personal
  - learning
  - sociology
---

TEXT TO TAG:
{text}
---

SUITABLE TAGS YAML:
```

## Performance

I wrote the output to a yaml file to inspect the results, over a thousand files tagged. Examples:

```yaml
- path: <obsidian_vault>/journals/20130000000000-galway.md
  tags:
    - personal
    - irish
    - history
- path: <obsidian_vault>/journals/daily/2019-06-01.md
  tags:
    - personal
    - writing
- path: <obsidian_vault>/notes/20210611120732-autonomy_mastery_and_purpose_framework.md
  tags:
    - psychology
    - career
    - workflow
    - leadership
```

These tags seem reasonably accurate, (quick tests with a small sample of already tagged notes saw an f1-score of 0.71), which is acceptable for my use-case.
