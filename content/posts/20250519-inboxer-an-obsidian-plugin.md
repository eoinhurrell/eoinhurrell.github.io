+++
title = "Inboxer: A Plugin for Obsidian"
author = ["Eoin H"]
publishDate = 2025-05-19T00:00:00+00:00
tags = ["post", "project"]
draft = false
+++

I'm excited to announce that [Inboxer](https://github.com/eoinhurrell/obsidian-inboxer), a small Obsidian plugin I created for my own workflow, is now available in Obsidian through Community Plugins.

As an ML engineer, I spend a lot of time in Obsidian managing research notes, and I kept running into an annoying workflow issue.

## The Problem

I'd found there was a lot of friction when trying to add details to a note, say a project or an idea, in a structured way. I'm in the habit of keeping an 'INBOX' heading where I can append random thoughts, links, websites etc, and for projects a 'TIMELINE' heading to record events and progress. It became annoying switching to a note, finding the heading and adding a subheading whenever I had new information. Context switching kills my flow, and those small interruptions add up.

## My Fix

So I built Inboxer. It's super simple - just two commands:

- **Add to Inbox**: Hit the shortcut, and boom - new heading under your INBOX section
- **Add to Timeline**: Same thing but adds a timestamped heading under TIMELINE automatically

Nothing fancy, but it's saved me tons of mental overhead. I just hit the shortcut, brain-dump my thought, and get back to work.

## How I Use It

I keep a central note for every project I work on, and frequently add items to the inbox like useful links, resources, or info that might help on the project. I also record updates in the project timeline.

I keep a single "Daily Note" open where I do most of my thinking. When a random idea strikes during my workday, I hit Alt+I (my shortcut for Add to Inbox), type it out, and continue with whatever I was doing. For my work journal entries, Alt+T adds a timestamped entry.

The plugin is smart enough to find existing INBOX/TIMELINE sections or create them if needed. It also respects heading hierarchy, which was important to me. You can customize the section names in settings if "INBOX" and "TIMELINE" aren't your style.

## Try It If You Want

Sometimes the smallest tools make the biggest difference in your workflow. Inboxer is available under Community Plugins in Obsidian, or you can install it manually from [GitHub](https://github.com/eoinhurrell/obsidian-inboxer). If it sounds like it would help, give it a go and me know if you find it useful.
