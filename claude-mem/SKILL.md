---
name: claude-mem
description: Long-term persistent memory system for Claude Code. Auto-captures decisions, learnings, and context across sessions using SQLite + vector search. Never lose context again. Use when you need to persist information across sessions, recall past decisions, or build up project knowledge over time.
---

# Claude-Mem

Persistent long-term memory across Claude Code sessions. Automatically captures what you do during sessions, compresses it with AI, and injects relevant context into future sessions.

## How It Works

1. **Session Start** — recalls relevant memories from past sessions
2. **During Session** — captures decisions, learnings, patterns
3. **Session End** — compresses and stores new memories
4. **Future Sessions** — relevant context auto-injected

## Storage

- SQLite + Chroma vector database at `~/.claude-mem/`
- Web viewer at `http://localhost:37777`
- Progressive three-layer retrieval: search → timeline → observations

## Commands

- `/mem:search <query>` — Search memories
- `/mem:timeline` — View memory timeline
- `/mem:status` — Check memory system status
