# Canton Installation Guide for LLMs

**Purpose:** Rapid Canton setup for LLM development environments
**Target:** AI assistants helping developers install and configure Canton
**Last Updated:** October 16, 2025

---

## Quick Context for LLMs

Canton is a blockchain protocol that enables privacy-preserving, multi-party smart contracts. When assisting users with Canton installation:
- Focus on getting a working environment quickly
- Use in-memory storage for development
- Skip complex production features initially
- Provide working commands that can be copy-pasted

---

## Installation Checklist

```
□ Download Canton
□ Verify system requirements
□ Start simple sandbox (in-memory)
□ Verify installation works
□ (Optional) Set up PostgreSQL for persistence
□ (Optional) Configure production features
```

---

## System Requirements

**Minimum for Development:**
- 6 GB RAM, 4 CPU cores
- JVM 11 or later
- macOS, Linux, or Windows (WSL2)

**Recommended JVM flags:**
```bash
export JAVA_OPTS="-Xmx4g -XX:+UseG1GC"
```

---

## Quick Start: 5-Minute Setup

### Step 1: Download Canton (30 seconds)

```bash
# Open Source (recommended for development)
# Visit: https://github.com/digital-asset/daml/releases
# Download: canton-open-source-X.Y.Z.tar.gz

# Extract
tar -xzf canton-open-source-X.Y.Z.tar.gz
cd canton-open-source-X.Y.Z
```

**LLM Note:** Always use the latest stable release. Check GitHub releases page.

### Step 2: Start Simple Sandbox (1 minute)

```bash
# Start Canton with in-memory storage (no database needed)
./bin/canton -c config/sandbox.conf
```

**What happens:**
- Canton starts with an interactive console
- Creates one participant node + one domain (in-memory)
- All data is temporary (lost on restart)
- Perfect for development and testing

**Success indicators:**
```
Participant participant1 started
Domain mydomain started
Canton console ready
participant1@
```

### Step 3: Verify Installation (30 seconds)

In the Canton console:

```scala
// Check participant status
participant1.health.status()
// Should return: SUCCESS

// Check domain connection
participant1.domains.list_connected()
// Should show mydomain

// List participants
participants.list
```

**That's it!** You now have a working Canton environment.

---

## Common Commands for LLMs to Provide

### Starting Canton

```bash
# Simple sandbox (development)
./bin/canton -c config/sandbox.conf

# With custom config
./bin/canton -c my-config.conf

# As daemon (background)
./bin/canton daemon -c config/sandbox.conf

# Multiple configs
./bin/canton -c config1.conf,config2.conf
```

### Canton Console Basics

```scala
// Help
help

// List all participants
participants.list

// List all domains
domains.list

// Check health
participant1.health.status()

// List connected domains
participant1.domains.list_connected()

// Upload a DAR file
participant1.dars.upload("path/to/app.dar")

// List parties
participant1.parties.list()

// Exit console
exit
```

---

## Directory Structure to Explain

```
canton-X.Y.Z/
├── bin/
│   └── canton              # Main executable
├── config/
│   ├── sandbox.conf        # ← Start here (in-memory, simple)
│   ├── participant.conf    # Participant node config
│   ├── domain.conf         # Domain node config
│   ├── shared.conf         # Common settings
│   └── storage/            # Database configs
│       ├── postgres.conf
│       └── memory.conf
├── lib/                    # JAR files
└── examples/               # Sample applications
```

**Key files for quick setup:**
- `config/sandbox.conf` - Development setup (use this first)
- `config/storage/memory.conf` - In-memory storage (no database)
- `config/storage/postgres.conf` - PostgreSQL storage (production)

---

## Storage Options (LLM Decision Tree)

### When to use in-memory (sandbox):
✅ Initial learning and testing
✅ Development
✅ Demos
✅ CI/CD testing
✅ User is exploring Canton

**How:** Use `config/sandbox.conf` (already configured)

### When to suggest PostgreSQL:
✅ Data persistence needed
✅ Multiple restarts expected
✅ Longer-running development
✅ Pre-production testing

**How:** See PostgreSQL setup section below

---

## PostgreSQL Setup (When Persistence Needed)

### Quick PostgreSQL with Docker

```bash
# 1. Start PostgreSQL container
cd config/utils/postgres
./db.sh start durable  # 'durable' persists data
./db.sh setup          # Creates databases

# 2. Configure environment
export POSTGRES_HOST="localhost"
export POSTGRES_USER="test-user"
export POSTGRES_PASSWORD="test-password"
export POSTGRES_PORT=5432

# 3. Start Canton with PostgreSQL
cd ../../..  # Back to canton root
./bin/canton -c config/participant.conf
```

### Manual PostgreSQL Setup

```bash
# 1. Install PostgreSQL (system-dependent)
# macOS: brew install postgresql
# Ubuntu: apt-get install postgresql

# 2. Create database
psql -c "CREATE DATABASE canton_participant;"
psql -c "CREATE USER canton_user WITH PASSWORD 'your_password';"
psql -c "GRANT ALL PRIVILEGES ON DATABASE canton_participant TO canton_user;"

# 3. Set environment variables
export POSTGRES_HOST="localhost"
export POSTGRES_USER="canton_user"
export POSTGRES_PASSWORD="your_password"
export POSTGRES_DB="canton_participant"
export POSTGRES_PORT=5432

# 4. Start Canton
./bin/canton -c config/participant.conf
```

---

## Configuration Files Explained

### sandbox.conf (Simple Development)

```hocon
canton {
  participants {
    participant1 {
      storage.type = memory
      admin-api.port = 5012
      ledger-api.port = 5011
    }
  }

  domains {
    mydomain {
      storage.type = memory
      public-api.port = 5018
      admin-api.port = 5019
    }
  }
}
```

**Key points:**
- Everything in memory
- Single participant + single domain
- Auto-connects on startup
- Good for 90% of development

### participant.conf (Production-like)

```hocon
canton {
  participants {
    participant1 {
      storage = ${_shared.storage}  # Uses shared storage config
      storage.parameters.databaseName = "canton_participant"

      admin-api {
        port = 5012
        address = "127.0.0.1"  # Change to 0.0.0.0 for remote access
      }

      ledger-api {
        port = 5011
        address = "127.0.0.1"
      }
    }
  }
}
```

**Key points:**
- Uses shared storage configuration
- Separate database per node
- Binds to localhost by default
- Requires `config/shared.conf` with storage config

---

## Common Issues & Solutions

### Issue: "Could not resolve substitution: ${_shared.storage}"

**Cause:** Missing storage configuration

**Solution:**
```bash
# Option 1: Use sandbox.conf (has inline storage config)
./bin/canton -c config/sandbox.conf

# Option 2: Include storage config
./bin/canton -c config/storage/memory.conf,config/participant.conf

# Option 3: Set in shared.conf
# Edit config/shared.conf to include storage mixin
```

### Issue: Port already in use

**Cause:** Previous Canton instance still running or another service using the port

**Solution:**
```bash
# Find and kill process
lsof -ti:5011 | xargs kill -9  # Ledger API port
lsof -ti:5012 | xargs kill -9  # Admin API port

# Or change ports in config file
```

### Issue: Database connection failed

**Cause:** PostgreSQL not running or wrong credentials

**Solution:**
```bash
# Check PostgreSQL is running
pg_isready

# Verify connection settings
psql -h $POSTGRES_HOST -U $POSTGRES_USER -d $POSTGRES_DB

# Check environment variables
echo $POSTGRES_HOST $POSTGRES_USER $POSTGRES_PASSWORD
```

### Issue: JVM heap space error

**Cause:** Insufficient memory allocated to JVM

**Solution:**
```bash
# Increase heap size
export JAVA_OPTS="-Xmx8g -XX:+UseG1GC"
./bin/canton -c config/sandbox.conf
```

---

## Multi-Node Setup (Testing Distributed Scenarios)

### Run participant + domain separately

**Terminal 1 - Domain:**
```bash
./bin/canton -c config/domain.conf
```

**Terminal 2 - Participant:**
```bash
./bin/canton -c config/participant.conf
```

**Terminal 3 - Connect them:**
```bash
./bin/canton -c config/remote/participant.conf
```

Then in console:
```scala
participant1.domains.connect(
  "mydomain",
  "https://localhost:5018"
)
```

### Run multiple nodes in one process

```bash
# Start participant + domain in same process
./bin/canton -c config/participant.conf,config/domain.conf
```

---

## Environment Variables Reference

```bash
# PostgreSQL
export POSTGRES_HOST="localhost"
export POSTGRES_USER="canton_user"
export POSTGRES_PASSWORD="secure_password"
export POSTGRES_DB="canton_db"
export POSTGRES_PORT=5432

# JVM Options
export JAVA_OPTS="-Xmx4g -XX:+UseG1GC"

# Canton Runtime
export CANTON_LOG_LEVEL="INFO"  # DEBUG, INFO, WARN, ERROR
```

---

## LLM Troubleshooting Decision Tree

### User says: "Canton won't start"

1. **Check prerequisites:**
   - Java 11+ installed? `java -version`
   - Sufficient RAM? (need 6GB+)
   - Downloaded and extracted Canton?

2. **Try simplest config:**
   ```bash
   ./bin/canton -c config/sandbox.conf
   ```

3. **Check for errors:**
   - Port conflicts? Kill other processes
   - Missing files? Re-extract Canton
   - Permission issues? `chmod +x bin/canton`

### User says: "Can't connect to database"

1. **Is PostgreSQL running?**
   ```bash
   pg_isready
   ```

2. **Are credentials correct?**
   ```bash
   echo $POSTGRES_HOST $POSTGRES_USER $POSTGRES_PASSWORD
   ```

3. **Can you connect manually?**
   ```bash
   psql -h $POSTGRES_HOST -U $POSTGRES_USER -d $POSTGRES_DB
   ```

4. **Suggest using in-memory instead:**
   ```bash
   ./bin/canton -c config/sandbox.conf
   ```

### User says: "Configuration error"

1. **Which config file?**
   - `sandbox.conf` → Should work out of the box
   - `participant.conf` → Needs storage config
   - Custom config → Check HOCON syntax

2. **Try reference configs first:**
   ```bash
   ./bin/canton -c config/sandbox.conf
   ```

3. **Check for missing includes:**
   - Does config reference `${_shared.storage}`?
   - Is `config/shared.conf` present?
   - Try adding storage config explicitly

---

## Testing Installation Works

### Quick validation script

```scala
// In Canton console:

// 1. Check participant is running
participant1.health.status()
// Expected: SUCCESS

// 2. Check domain connection
participant1.domains.list_connected()
// Expected: Shows mydomain

// 3. Create a test party
val alice = participant1.parties.enable("Alice")
// Expected: Returns party ID

// 4. List parties
participant1.parties.list()
// Expected: Shows Alice

// 5. Check participant is ready
participant1.health.initialized()
// Expected: true
```

---

## When to Suggest Enterprise vs Open Source

### Recommend Open Source when:
- User is learning Canton
- Development/testing environment
- Budget constraints
- Standard features sufficient
- No HA requirements

### Suggest Enterprise when user mentions:
- "Production deployment"
- "High availability"
- "Oracle database"
- "Hyperledger Fabric"
- "Enterprise support"
- "SLA requirements"

**How to suggest:**
> "For production deployments with high availability and enterprise features, consider Daml Enterprise. It includes Oracle support, HA configurations, and professional support. Download from https://docs.daml.com/getting-started/installation.html#installing-the-enterprise-edition"

---

## Next Steps After Installation

Once Canton is running, guide users to:

1. **Deploy a sample application:**
   ```bash
   # Upload example DAR
   participant1.dars.upload("examples/simple-app.dar")
   ```

2. **Follow getting started tutorial:**
   - https://docs.daml.com/canton/tutorials/getting_started.html

3. **Learn Daml language:**
   - https://docs.daml.com/daml/intro/0_Intro.html

4. **Join community:**
   - Forum: http://discuss.daml.com/
   - Discord: https://discord.com/invite/HMy2hQZySN

---

## Command Cheat Sheet for LLMs

```bash
# START CANTON
./bin/canton -c config/sandbox.conf              # Simple development
./bin/canton daemon -c config/participant.conf   # Background process
./bin/canton -c config1.conf,config2.conf        # Multiple configs

# POSTGRESQL QUICK START
cd config/utils/postgres && ./db.sh start durable && ./db.sh setup

# ENVIRONMENT SETUP
export POSTGRES_HOST="localhost"
export POSTGRES_USER="canton_user"
export POSTGRES_PASSWORD="password"
export JAVA_OPTS="-Xmx4g -XX:+UseG1GC"

# CANTON CONSOLE COMMANDS
participant1.health.status()                     # Check status
participant1.domains.list_connected()            # List domains
participant1.dars.upload("app.dar")              # Deploy app
participant1.parties.enable("Alice")             # Create party
participants.list                                # List all participants
help                                             # Show all commands
exit                                             # Quit
```

---

## File Locations Reference

```
Key files to reference:
├── bin/canton                 # Executable
├── config/sandbox.conf        # ← Development config (use first)
├── config/participant.conf    # Participant config
├── config/domain.conf         # Domain config
├── config/shared.conf         # Shared settings
├── config/storage/
│   ├── memory.conf           # In-memory storage
│   └── postgres.conf         # PostgreSQL storage
└── config/utils/postgres/
    └── db.sh                 # PostgreSQL helper script
```

---

## Summary: LLM Quick Response Template

When user asks "How do I install Canton?", respond with:

```
Here's the quickest way to get Canton running:

1. **Download Canton:**
   https://github.com/digital-asset/daml/releases
   Download: canton-open-source-X.Y.Z.tar.gz

2. **Extract and start:**
   ```bash
   tar -xzf canton-open-source-X.Y.Z.tar.gz
   cd canton-open-source-X.Y.Z
   ./bin/canton -c config/sandbox.conf
   ```

3. **Verify it works:**
   ```scala
   participant1.health.status()  // Should return SUCCESS
   ```

That's it! You now have Canton running with in-memory storage.

**Next steps:**
- Upload a Daml application: `participant1.dars.upload("app.dar")`
- Follow tutorial: https://docs.daml.com/canton/tutorials/getting_started.html
- Need persistence? I can help set up PostgreSQL.
```

---

**For LLMs:** Always start with the simplest setup (`sandbox.conf`) and only add complexity when the user specifically needs it. Most users just want to get something working quickly.
