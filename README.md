# 📁 RevRAG.ai — Work Documentation
 
> A living record of features, bug fixes, and optimizations worked on during my time at **RevRAG.ai**.  
> Each entry is documented progressively — updated as work happens, not after.
 
---
 
## 📌 About This Repository
 
This repository serves as my personal technical journal for all engineering work done at RevRAG.ai. Every feature built, bug squashed, or system optimized gets its own dedicated markdown file — written and updated in real time as the work progresses.
 
The goal is to maintain a clear, honest, and detailed log that captures:
- **What** was worked on
- **Why** it was needed
- **How** it was approached and implemented
- **What** the outcome was
---
 
## 🗂️ Structure
 
```
/
├── README.md               ← You are here
├── features/               ← New feature documentation
│   └── <title>.md
├── bugfixes/               ← Bug investigation & fix docs
│   └── <title>.md
└── optimizations/          ← Performance & refactor docs
    └── <title>.md
```
 
Each file is named by a short descriptive title. The branch name, date, and full context live inside the document.
 
---
 
## 📋 Documentation Format
 
Every entry follows a consistent structure:
 
```
# [Type] Title
 
| Field | Details |
Branch, Type, Status, Started, Last Updated
 
## Overview
## Background / Context
## Call Flow Analysis / Root Cause
## Identified Delay Sources / Bug Details
## Open Questions
## Approach
## Implementation Details
## Testing
## Outcome / Results
## Notes / Learnings
```
 
---
 
## 📚 Index
 
### ✨ Features
| Title | Branch | Started | Status |
|-------|--------|---------|--------|
| [Independent Fallback Adapter Chains (LLM / TTS / STT)](./features/model-fallback-feature.md) | Titiksha/model-fallback-clean · Titiksha/model-fallback | 2026-04-15 | Completed |
 
### 🐛 Bug Fixes
| Title | Branch | Started | Status |
|-------|--------|---------|--------|
| [Call History Duration Filter Empties Table](./bugfixes/call-history-duration-filter-bug.md) | bug-fix | 2026-04-16 | Completed |
 
### ⚡ Optimizations
| Title | Branch | Started | Status |
|-------|--------|---------|--------|
| [Reduce Caller Wait Time](./optimizations/caller-wait-time-optimization.md) | Titiksha/agent-start-optimization | 2026-04-16 | In Progress |
 
---
 
## 🔄 Workflow
 
1. **Start of work** — Create a new `.md` file under the relevant folder, named by a short descriptive title.
2. **During work** — Update the file as decisions are made, blockers are hit, and progress is made.
3. **On completion** — Finalize the doc with outcomes, results, and any retrospective notes.
4. **Update index** — Add/update the entry in the table above with status marked as `Completed`.
---
 
## 🏢 Context
 
**Company:** revrag.ai  
**Role:** *SDE Intern*  
**Duration:** *19 March, 2026 – present*
 
---
 
*This documentation is maintained for personal reference, learning, and portfolio purposes.*
