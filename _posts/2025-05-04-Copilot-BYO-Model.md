---
layout: post
title: Bring Your Own Model with GitHub Copilot
date: 2025-05-04 10:00:00-0400
description: >
  GitHub Copilot is a very helpful tool for developers, and one that I use all the time in my daily operations.  One thing clients ask me a lot is about the different models that are available with Copilot, and they've wondered in the past if, instead of using the ones provided by GitHub, they could bring their own.  In this post, I'll walk through how you can bring your own model to GitHub Copilot, and some of the things you should be aware of when doing so.
img: copilot-byo-model/hero.png
tags: [GitHub, Copilot, GitHub Copilot, AI]
---

I use GitHub Copilot all the time in my day-to-day tasks.  In fact, in a [previous post](https://blog.matt-o.com/Copilot-Vscode-Chat-Extension) I spent a day during a hackathon to write a chat extension for Visual Studio Code which allowed Copilot to better understand a SQL Database.  Now that MCP is a thing, that's a bit less relevant.

One thing that I get asked a lot is "If GitHub Copilot isn't limited to OpenAI models anymore, why can't we provide our own AI models?"  It's an interesting question, and one that I'd also been wondering about for a while.  Well, about a month ago, I stumbled upon a [LinkedIn Post by Rory Preddy](https://www.linkedin.com/posts/rorypreddy_programming-development-ai-activity-7309461000963997696-4-f-/?utm_source=share&utm_medium=member_desktop&rcm=ACoAAAEw4MkBGAZ2386PMTS1nK_fXBy-oLOcfGo) which caught my attention.  It had two main statements in it: "🎉 Now in preview 👉Bring Your Own Key/Model in VS Code Insiders" and, more importantly, "run Ollama models."  I use Ollama for local LLM experimentation with models like [codegemma](https://ai.google.dev/gemma/docs/codegemma) and [codellama](https://github.com/meta-llama/codellama/tree/main) and [codestral](https://mistral.ai/news/codestral), lately, [phi4](https://azure.microsoft.com/en-us/products/phi/).  If I could combine these Ollama models with Copilot, how well would they work?  Well, [I decided to find out](https://github-copilot.xebia.ms/detail?videoId=46).

<iframe width="560" height="315" src="https://www.youtube.com/embed/hFM9gtIzBMI?si=Ezk59XTVFOeCzVVn" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

All in all, I'd say this feature is great.  It was really easy to set up, and overall I've been impressed with how well the other models are performing on my local machine.  While I've got nothing against the [models provided by GitHub](https://docs.github.com/en/enterprise-cloud@latest/copilot/using-github-copilot/ai-models/choosing-the-right-ai-model-for-your-task) it was really interesting to see how different models reacted to various prompts and tasks from Copilot.  This feature, combined with [using MCP services with Copilot](https://github-copilot.xebia.ms/detail?videoId=45), could really boost GitHub Copilot's usefulness by empowering it with the right services and models for the task at hand.
