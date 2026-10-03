---
layout: post
title: "Hello, world — what this blog is for"
tags: [meta]
---

This is my first post. I'll use this space to write up the things I build and learn: LLM infrastructure, retrieval, agents, and whatever I'm digging into on the side.

## What to expect

- **Build logs** — how a project came together, what broke, what I'd do differently.
- **Deep dives** — one concept, explained properly.
- **Notes** — shorter things worth writing down.

## A code block, to check highlighting works

```python
def cache_key(prompt: str, model: str) -> str:
    # exact-match key; a semantic cache would embed the prompt instead
    return f"{model}:{hash(prompt)}"
```

> Delete this post (or rewrite it) once you've published your first real one.
