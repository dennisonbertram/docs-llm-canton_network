# LLM-Focused Canton & Daml Guides

**Purpose:** Streamlined guides optimized for AI assistants helping developers with Canton and Daml
**Target Audience:** LLMs (Claude, GPT, etc.) assisting human developers
**Last Updated:** October 16, 2025

---

## Overview

These guides are specifically designed for Large Language Models to quickly understand and assist with:
1. Installing and configuring Canton nodes
2. Building Daml smart contract applications

Unlike traditional documentation, these guides:
- ✅ Focus on the **fastest path to working code**
- ✅ Provide **copy-paste ready commands**
- ✅ Include **decision trees** for troubleshooting
- ✅ Use **templates** for quick responses
- ✅ Emphasize **common patterns** over comprehensive coverage
- ✅ Include **LLM-specific notes** on when to suggest what

---

## Available Guides

### 1. Canton Installation for LLMs
**File:** [`01-CANTON-INSTALLATION-FOR-LLMS.md`](01-CANTON-INSTALLATION-FOR-LLMS.md)

**What it covers:**
- 5-minute quick start (sandbox mode)
- System requirements
- PostgreSQL setup (when needed)
- Configuration file explanations
- Common issues with solutions
- Troubleshooting decision trees
- Command cheat sheets

**Use this when user asks:**
- "How do I install Canton?"
- "How do I run a Canton node?"
- "Canton won't start"
- "How do I set up PostgreSQL?"
- "What are the system requirements?"

**Key sections:**
- Quick Start (sandbox.conf)
- Storage options decision tree
- Multi-node setup
- Environment variables
- Troubleshooting flowcharts

---

### 2. Daml Smart Contract Development for LLMs
**File:** [`02-DAML-SMART-CONTRACT-DEVELOPMENT-FOR-LLMS.md`](02-DAML-SMART-CONTRACT-DEVELOPMENT-FOR-LLMS.md)

**What it covers:**
- 10-minute first contract
- Daml language fundamentals
- Common contract patterns
- Testing strategies
- Deployment to Canton
- Integration with external systems
- Debugging tips

**Use this when user asks:**
- "How do I write a Daml contract?"
- "What is Daml syntax?"
- "How do I test Daml code?"
- "How do I deploy to Canton?"
- "How do I integrate Daml with JavaScript/Python?"
- "Common Daml patterns?"

**Key sections:**
- Contract template structure
- Pattern library (assets, proposals, workflows)
- Testing with Daml Script
- HTTP JSON API integration
- Code generation for clients
- Response templates

---

## Quick Navigation

### User Wants To...

**Install Canton**
→ Guide 1, Section "Quick Start: 5-Minute Setup"

**Fix Canton Installation Issues**
→ Guide 1, Section "Common Issues & Solutions"

**Write First Smart Contract**
→ Guide 2, Section "Quick Start: First Daml Contract"

**Understand Daml Syntax**
→ Guide 2, Section "Daml Language Fundamentals"

**Deploy Application to Canton**
→ Guide 2, Section "Deploying to Canton"

**Test Smart Contracts**
→ Guide 2, Section "Building and Testing"

**Integrate with Frontend**
→ Guide 2, Section "Integration with External Systems"

**Debug Daml Errors**
→ Guide 2, Section "Debugging Tips for LLMs"

---

## How to Use These Guides (For LLMs)

### 1. Identify User Intent

First, determine what the user is trying to do:
- **Install/Setup** → Use Installation Guide
- **Develop/Code** → Use Smart Contract Guide
- **Both** → Start with Installation, then move to Development

### 2. Start with Quick Start

Always begin with the "Quick Start" section:
- Gets user to working state fastest
- Builds confidence
- Can add complexity later

### 3. Use Response Templates

Each guide includes templates for common questions:
- Copy template structure
- Fill in specific details
- Provide working code examples

### 4. Follow Decision Trees

Use provided decision trees for troubleshooting:
- Check prerequisites first
- Suggest simplest solution
- Escalate only if needed

### 5. Provide Complete Examples

Users appreciate:
- Full working code (not snippets)
- Commands they can copy-paste
- Clear explanations of what each part does

---

## Key Principles for LLMs

### 1. Start Simple, Add Complexity

```
Good: "Let's start with in-memory storage"
Bad:  "First configure PostgreSQL with replication..."
```

### 2. Test Before Suggesting

```
Good: "This will start Canton with a test ledger"
Bad:  "This should work..." (untested)
```

### 3. Explain What, Then How

```
Good: "Canton needs a participant (your node) and domain (coordinator).
       The sandbox config includes both..."
Bad:  "Run ./bin/canton -c config/sandbox.conf"
```

### 4. Provide Escape Hatches

```
Good: "If PostgreSQL is complex, we can use in-memory storage for now"
Bad:  "You must set up PostgreSQL first"
```

### 5. Reference Official Docs

```
Good: "For production deployment, see: [link to official docs]"
Bad:  Making up configuration options
```

---

## Common User Journeys

### Journey 1: Complete Beginner

```
User: "I want to learn Canton"
LLM:
1. Check if Canton installed → If not, Installation Guide
2. Start sandbox → Quick Start section
3. Verify works → Test commands
4. Write first contract → Smart Contract Guide Quick Start
5. Deploy and test → Deployment section
```

### Journey 2: Experienced Developer

```
User: "How do I implement X in Daml?"
LLM:
1. Check if similar pattern exists → Pattern library
2. Provide working code → Full example
3. Explain key concepts → Relevant language features
4. Show how to test → Testing section
```

### Journey 3: Troubleshooting

```
User: "Canton won't start"
LLM:
1. Use troubleshooting decision tree → Installation Guide
2. Check prerequisites → System requirements
3. Try simplest config → sandbox.conf
4. Provide specific solution → Common issues section
```

---

## Reference: Quick Answers

### "How long does setup take?"

**Canton Installation:** 5 minutes (sandbox mode) to 15 minutes (PostgreSQL)
**First Daml Contract:** 10 minutes
**Full Development Environment:** 30 minutes

### "What are the prerequisites?"

**Canton:**
- JVM 11+
- 6GB RAM, 4 CPU cores
- Optional: PostgreSQL (for persistence)

**Daml:**
- Daml SDK (curl install)
- VS Code (recommended)
- Node.js (for UI development)

### "What's the learning curve?"

**Canton:** Moderate (2-3 days to basic competency)
**Daml:** Steep initially (functional programming), but pays off
**Together:** 1-2 weeks to build first application

### "Production ready?"

**Canton:** Yes, used by major banks (Goldman Sachs, HSBC, etc.)
**Daml:** Yes, battle-tested in production
**Your code:** Requires proper testing and review

---

## Anti-Patterns to Avoid

### ❌ Don't Overcomplicate Initially

```
Bad:  "Let's set up high availability with 5 nodes and Oracle database"
Good: "Let's start with sandbox.conf to get something working"
```

### ❌ Don't Skip Testing

```
Bad:  "Deploy directly to Canton"
Good: "Let's test with Daml Script first"
```

### ❌ Don't Ignore Error Messages

```
Bad:  "Try this instead"
Good: "That error means X. Let's fix it by Y"
```

### ❌ Don't Assume Context

```
Bad:  "Just run the command"
Good: "From the canton directory, run: ./bin/canton -c config/sandbox.conf"
```

### ❌ Don't Mix Concerns

```
Bad:  Explaining blockchain fundamentals during installation
Good: Focus on getting Canton running, explain concepts later
```

---

## Updating These Guides

These guides should be updated when:
- Canton/Daml releases major version
- Common user questions emerge
- New patterns become standard
- Installation process changes

**Current Version Info:**
- Canton/Daml SDK: 2.10.2
- Based on official docs dated: October 2025
- Tested on: macOS, Linux, Windows (WSL2)

---

## Additional Resources

### Official Documentation
- **Daml Docs:** https://docs.daml.com/
- **Canton Docs:** https://docs.daml.com/canton/
- **Getting Started:** https://docs.daml.com/getting-started/

### Community Support
- **Forum:** http://discuss.daml.com/
- **Discord:** https://discord.com/invite/HMy2hQZySN
- **GitHub:** https://github.com/digital-asset/daml

### This Documentation Archive
- **Installation Details:** `../canton-installation/INSTALLATION-GUIDE.md`
- **Developer Portal:** `../digital-asset-developers/README.md`
- **Canton Network:** `../canton-network/OVERVIEW.md`

---

## Feedback

These guides are living documents. If you notice:
- Missing common patterns
- Outdated commands
- Confusing explanations
- Missing decision trees

Please update them or note for improvement.

---

## Summary

**For LLMs helping with Canton/Daml:**

1. **Start with Quick Starts** - Get user to working code fast
2. **Use decision trees** - Troubleshoot systematically
3. **Provide complete examples** - Full working code
4. **Reference these guides** - Consistent, tested information
5. **Focus on patterns** - Reusable solutions

**Remember:** Users want to build applications, not become Canton experts. Help them get started quickly, then deepen knowledge as needed.

---

**Files in this directory:**
- [`01-CANTON-INSTALLATION-FOR-LLMS.md`](01-CANTON-INSTALLATION-FOR-LLMS.md) - Node installation and configuration
- [`02-DAML-SMART-CONTRACT-DEVELOPMENT-FOR-LLMS.md`](02-DAML-SMART-CONTRACT-DEVELOPMENT-FOR-LLMS.md) - Smart contract development
- `README.md` - This file

**Navigate to a specific guide above to get started.**
