# OpenClaw Memory Management Guide

## Why Your Agent Has Amnesia

OpenClaw's default memory setup is a junk drawer that costs you money. The default configuration dumps everything into one massive memory file, which confuses the agent and bloats your API bill.

When your agent reads one giant file for every request, you burn through tokens and the bot gets confused. Most people accept this and waste tokens by repasting context and reexplaining their project in every session.

It doesn't have to be this way.

This guide presents four methods to give your OpenClaw agent real persistent memory—from the dead simple to the seriously powerful. The last method is one that most people don't even know OpenClaw is capable of.

---

## Understanding Context Windows

Before diving into solutions, let's understand the problem.

OpenClaw agents (and all LLM-based agents) operate within a context window. Think of it as short-term memory. Whatever is in that window, the agent can see. But the second a session ends, that window closes and the context is gone.

For one-off conversations, this is fine. But if you're using OpenClaw as an ongoing collaborator—a coding partner or research assistant—you need to remember things between sessions:

- What project you're working on
- Your preferences
- Decisions you've already made
- Data the agent has gathered

Here are four ways to solve this problem.

---

## Method 1: Structured Memory Folders

**Complexity:** Simple  
**Setup Time:** ~60 seconds  
**Best For:** Getting started fast, full transparency

### The Concept

Create a dedicated folder structure in your project directory specifically designed to store context that persists between sessions.

### Recommended Structure

```
memory/
├── projects/
│   └── project-context/
│       ├── goals.md
│       ├── decisions.md
│       └── current-status.md
├── preferences/
│   ├── coding-style.md
│   └── communication-preferences.md
└── knowledge-base/
    ├── research-notes.md
    └── references.md
```

### How to Use It

The key is not just creating these folders—you must instruct the agent to use them.

In your custom instructions or system prompt, tell your OpenClaw agent:

> "At the end of every session, update the relevant files in the memory folder."

That's it. That's the whole trick.

### Why This Works

1. **Transparency:** You can open those Markdown files and see exactly what your agent remembers. No black box.
2. **Provider Agnostic:** Works with any LLM provider—Anthropic, OpenAI, local models. It's just files.
3. **Control:** You decide what's important enough to remember. This intentional memory design leads to better results than dumping everything into a database and hoping retrieval works.

### Limitations

- Not sophisticated
- No vector database or semantic search
- Manual structure design required

For most people, especially if you're just getting started, this is the best first step.

---

## Method 2: Memory Search (Built-in)

**Complexity:** Medium  
**Setup Time:** ~10 minutes  
**Best For:** Natural language recall with proper setup

### The Concept

OpenClaw has a built-in memory search function designed to let your agent search through stored memories, past conversations, and saved context.

When it works, it's powerful. You can say:
- "Remember that I prefer TypeScript over JavaScript."
- "Remember that the database schema uses unique primary IDs as keys."

Later, your agent can search for and retrieve those memories.

### The Critical Requirement

**Memory Search requires OpenAI, Gemini, or Voyage API to function and is turned off by default.**

This is the part that trips people up. Memory Search uses OpenAI, Gemini, or Voyage's embedding models under the hood.

**If you're only running Anthropic's Claude as your LLM provider, memory search will silently fail or not work at all.** You won't always get a clear error message—your agent will just not remember things.

### Setup Instructions

Even if Claude is your primary model for conversations, you need an OpenAI, Voyage, or Gemini API key configured in your OpenClaw setup specifically for the embedding layer that powers memory search.

1. Go into your OpenClaw settings
2. Find the API configuration section
3. Add a valid OpenAI, Voyage, or Gemini API key
4. Enable memory search
5. Configure it to use the model you added

The API key doesn't have to be your main model—it's just used for embedding and search functionality.

### Cost

Embedding calls are very cheap (fractions of a cent). Don't let the "I need another API key" concern scare you off.

### Summary

If Method 1 was the manual but reliable approach, Method 2 is the built-in but needs proper setup approach. This is a great solution for most users once you understand the API key requirement.

---

## Method 3: MEM0 Plugin (Third-Party)

**Complexity:** Medium-High  
**Setup Time:** ~20 minutes  
**Best For:** Automated long-term conversational memory with zero maintenance

### The Concept

MEM0 is a third-party plugin purpose-built to solve the persistent memory problem. It adds long-term memory to OpenClaw agents by automatically watching conversations, extracting what matters, and bringing it back when relevant.

**Plugin:** `@mem0/openclaw-mem0`

### How It Works

MEM0 uses vector search. A vector database stores information not as exact text matches, but as mathematical representations of meaning.

When your agent needs to recall something, it performs semantic search. If you told your agent 3 weeks ago "our client prefers a minimalist design approach," and today you ask about design direction, MEM0 will surface that memory even though you used completely different words.

### The Three-Step Process

1. **Watch:** MEM0 observes every conversation you have with your OpenClaw agent in the background
2. **Extract:** It identifies information that matters (preferences, decisions, facts, project details) and stores them as vector embeddings
3. **Retrieve:** At the start of each conversation or whenever context is relevant, it pulls back the right memories and injects them into your agent's context

All of this happens automatically with no manual work from you.

### Comparison to Method 1

- **Method 1 (Structured Folders):** You decide what to save and design the retrieval
- **Method 3 (MEM0):** The system does the heavy lifting for you

### Trade-offs

- Third-party dependency
- You're potentially sending data through MEM0's infrastructure (read their privacy docs)
- May occasionally surface irrelevant memories or miss something you wanted it to catch

### Summary

For users who want a "set it and forget it" memory solution, MEM0 is the move. Once installed and configured, you just use OpenClaw normally. MEM0 handles the rest.

Have a conversation today, close the session, come back tomorrow—your agent actually remembers what happened.

---

## Method 4: SQLite Database (Native)

**Complexity:** High  
**Setup Time:** Variable  
**Best For:** Dense structured data that needs precise queries

### The Power Move

OpenClaw can natively read from and write to a SQLite database. You don't need a plugin. You don't need an extra API key. It's just there.

### Why This Is a Big Deal

Methods 1-3 are great for conversational memory, preferences, decisions, and general context. But what about dense structured data?

Think about:
- Hundreds of API endpoints you need documented
- Product catalogs
- Customer records
- Research data with dozens of fields
- Financial figures across multiple quarters

Markdown files fall apart with that kind of data. Vector search is great for fuzzy retrieval, but sometimes you need exact queries:

> "Show me every API endpoint that uses POST and requires authentication."

That's a SQL query. And SQL is really good at that.

### How It Works in Practice

You say to your OpenClaw agent:

> "I need you to create a SQLite database to track all the API endpoints in our project. Store the endpoint path, HTTP method, authentication requirement, rate limit, and a description for each one."

Your agent will:
1. Create the database
2. Define the schema
3. Start populating it

You can then query it naturally:
- "How many endpoints require authentication?"
- "Show me all GET endpoints related to user management."

Your agent translates that into SQL, runs it, and gives you the answer.

### The Full Memory Architecture

Here's where it gets really powerful: you can combine this with the other methods.

- **Structured folders:** High-level project context and preferences
- **MEM0 or Memory Search:** Conversational memory between sessions
- **SQLite:** Dense structured data that needs precise querying

This creates a full memory architecture:
- **Short-term memory:** Context window
- **Medium-term memory:** MEM0 and Memory Search
- **Long-term structured data:** SQLite

You've essentially built a brain.

### Additional Benefits

Because SQLite is just a file (a single `.db` file):
- It's portable
- It's persistent
- It survives session resets
- You can even version control it

---

## Summary: Choosing Your Method

You're not limited to one memory method. The best OpenClaw setups use two or three of these together, each handling a different type of information.

### Method Comparison

| Method | Complexity | Best For | Key Requirement |
|--------|-----------|----------|-----------------|
| **1. Structured Folders** | Simple | Getting started fast, full transparency | Works with any LLM |
| **2. Memory Search** | Medium | Quick natural language recall | OpenAI/Gemini/Voyage API key |
| **3. MEM0 Plugin** | Medium-High | Automated long-term conversational memory | Third-party plugin |
| **4. SQLite Database** | High | Dense structured data with precise queries | Native OpenClaw capability |

### Recommendation

1. **Start with Method 1 today** (takes ~5 minutes)
2. **Add MEM0 or SQLite** depending on your use case
3. **Configure Memory Search properly** with the correct API key

This layered approach gives you the best of all worlds: transparency, automation, and power.

---

## Next Steps

- Set up structured memory folders in your project
- Configure Memory Search with the appropriate API key
- Evaluate whether MEM0 or SQLite fits your use case
- Combine methods for a comprehensive memory architecture

Your OpenClaw agent will transform from having amnesia to having a robust, persistent memory system that actually works.