# Daml Smart Contract Development: Quick Start

**Goal:** Deploy a working Daml smart contract to Canton in 30 minutes
**Tested:** October 16, 2025 with Daml SDK 2.10.2
**System:** macOS (similar for Linux, Windows requires WSL)

---

## Prerequisites

Install these first:

```bash
# 1. Daml SDK (~15 minutes)
curl -sSL https://get.daml.com/ | sh

# 2. Java 11 (~30 seconds)
brew install openjdk@11
```

**Time budget:** 15-20 minutes for installation

---

## Step 1: Environment Setup

**Configure PATH for both tools:**

```bash
# Add to PATH (run in each new terminal OR add to ~/.zshrc)
export PATH="$HOME/.daml/bin:$PATH"
export PATH="/opt/homebrew/opt/openjdk@11/bin:$PATH"

# Verify installation
daml version    # Should show: 2.10.2
java -version   # Should show: openjdk version "11.0.28"
```

**Why Java is required:**
- Daml builds without Java
- Testing and script execution require Java
- Canton runs on JVM (requires Java)

**Time:** < 1 minute

---

## Step 2: Create Project

```bash
# Navigate to your workspace
cd ~/Develop/canton/

# Create new project
daml new my-first-contract

# Enter project
cd my-first-contract
```

**What this creates:**
- `daml/Main.daml` - Default Asset contract template
- `daml.yaml` - Project configuration
- `.gitignore` - Git exclusions (/.daml, /log)

**Time:** < 5 seconds

---

## Step 3: Create IOU Contract

Replace `daml/Main.daml` with this simple IOU (I Owe You) contract:

```daml
module Main where

import Daml.Script

-- Simple IOU contract
template Iou
  with
    issuer : Party    -- Who owes money
    owner : Party     -- Who is owed money
    amount : Decimal  -- How much
  where
    signatory issuer  -- Issuer must sign to create
    observer owner    -- Owner can see it

    -- Owner can transfer the IOU to someone else
    choice Transfer : ContractId Iou
      with
        newOwner : Party
      controller owner
      do
        create this with owner = newOwner

    -- Issuer can settle by paying
    choice Settle : ()
      controller issuer
      do
        return ()  -- Archive the contract

-- Test script
setup : Script ()
setup = script do
  -- Allocate parties
  alice <- allocatePartyWithHint "Alice" (PartyIdHint "Alice")
  bob <- allocatePartyWithHint "Bob" (PartyIdHint "Bob")
  charlie <- allocatePartyWithHint "Charlie" (PartyIdHint "Charlie")

  -- Create users
  aliceId <- validateUserId "alice"
  bobId <- validateUserId "bob"
  charlieId <- validateUserId "charlie"
  createUser (User aliceId (Some alice)) [CanActAs alice]
  createUser (User bobId (Some bob)) [CanActAs bob]
  createUser (User charlieId (Some charlie)) [CanActAs charlie]

  -- Alice creates IOU owing Bob $100
  iouId <- submit alice do
    createCmd Iou with
      issuer = alice
      owner = bob
      amount = 100.0

  -- Bob transfers IOU to Charlie
  newIouId <- submit bob do
    exerciseCmd iouId Transfer with newOwner = charlie

  -- Charlie verifies he owns it
  Some iou <- queryContractId charlie newIouId
  assert (iou.owner == charlie)
  assert (iou.amount == 100.0)

  -- Alice settles the IOU
  submit alice do
    exerciseCmd newIouId Settle

  return ()
```

**Contract explanation:**
- **Issuer:** Party who owes money (signatory - must approve creation)
- **Owner:** Party who is owed (observer - can see but doesn't sign)
- **Transfer:** Owner can assign IOU to someone else
- **Settle:** Issuer can pay off and archive the IOU

**Time:** 2-3 minutes to create file

---

## Step 4: Build and Test

```bash
# Build the contract
daml build
```

**Expected output:**
```
Compiling my-first-contract to a DAR.
Created .daml/dist/my-first-contract-0.0.1.dar
```

**Build time:** < 1 second

**Run tests:**
```bash
daml test
```

**Expected output:**
```
Test Summary

daml/Main.daml:setup: ok, 0 active contracts, 3 transactions.
- Internal templates: 1 defined, 1 (100.0%) created
- Internal template choices: 3 defined, 2 (66.7%) exercised
```

**Test results:**
- 3 transactions: create IOU, transfer, settle
- 0 active contracts (IOU was settled/archived)
- 100% template coverage
- 66.7% choice coverage (Transfer + Settle, Archive not needed)

**Time:** 2-3 seconds

---

## Step 5: Canton Configuration

Create Canton configuration file at `/tmp/canton-simple.conf`:

```hocon
canton {
  parameters {
    manual-start = no
  }

  participants {
    participant1 {
      storage.type = memory
      admin-api.port = 5012
      ledger-api.port = 5011
    }
  }

  domains {
    domain1 {
      storage.type = memory
      public-api.port = 5018
      admin-api.port = 5019
      init.domain-parameters.protocol-version = 5
    }
  }
}
```

**Key configuration points:**
- `manual-start = no` - Auto-start all nodes
- `storage.type = memory` - In-memory storage (dev mode, clears on restart)
- `ledger-api.port = 5011` - Most important port for client connections
- `protocol-version = 5` - **REQUIRED** - Matches Daml SDK 2.10.2

**Time:** < 1 minute

---

## Step 6: Bootstrap Script

Create bootstrap script at `/tmp/canton-bootstrap.canton`:

```scala
// Connect participant to domain
participant1.domains.connect_local(domain1)

// Allocate parties
val alice = participant1.parties.enable("Alice")
val bob = participant1.parties.enable("Bob")
val charlie = participant1.parties.enable("Charlie")

println(s"Allocated party Alice: $alice")
println(s"Allocated party Bob: $bob")
println(s"Allocated party Charlie: $charlie")
```

**What this does:**
- Connects participant to domain (establishes network)
- Allocates 3 parties on the participant
- Prints party IDs for verification

**Time:** < 1 minute

---

## Step 7: Start Canton

```bash
# Ensure Java in PATH
export PATH="/opt/homebrew/opt/openjdk@11/bin:$PATH"

# Start Canton daemon with bootstrap
nohup java -jar ~/.daml/sdk/2.10.2/canton/canton.jar \
  daemon \
  -c /tmp/canton-simple.conf \
  --bootstrap /tmp/canton-bootstrap.canton \
  > /tmp/canton.log 2>&1 &

CANTON_PID=$!
echo "$CANTON_PID" > /tmp/canton.pid
echo "Canton started with PID: $CANTON_PID"

# Wait for startup
sleep 30
```

**Command breakdown:**
- `daemon` - Run in background (no interactive console)
- `-c <config>` - Configuration file
- `--bootstrap <script>` - Run initialization script on startup
- `> /tmp/canton.log` - Log output to file
- `&` - Run as background process

**Startup time:** ~30 seconds

**Time:** ~40 seconds (including wait)

---

## Step 8: Verify Canton

```bash
# Check process running
ps aux | grep canton | grep -v grep

# Check ports listening
lsof -i :5011 -i :5012 | grep LISTEN

# Check parties allocated
grep "Allocated party" /tmp/canton.log
```

**Expected log output:**
```
Allocated party Alice: Alice::1220f5f0704a...
Allocated party Bob: Bob::1220f5f0704a...
Allocated party Charlie: Charlie::1220f5f0704a...
```

**Success indicators:**
- Canton process visible in `ps`
- Ports 5011 and 5012 are LISTEN
- All 3 parties allocated in log

**Time:** < 30 seconds

---

## Step 9: Upload DAR to Canton

```bash
# Ensure in project directory
cd ~/Develop/canton/my-first-contract

# Upload DAR to Canton
daml ledger upload-dar \
  .daml/dist/my-first-contract-0.0.1.dar \
  --host localhost \
  --port 5011
```

**Expected output:**
```
Uploading .daml/dist/my-first-contract-0.0.1.dar to localhost:5011
DAR upload succeeded.
```

**What happens:**
- DAR file uploaded to Canton participant
- All templates registered
- Contracts ready to be created

**Upload time:** 2-3 seconds

**Time:** < 10 seconds

---

## Step 10: Start JSON API (Optional)

```bash
# Start JSON API server
nohup daml json-api \
  --ledger-host localhost \
  --ledger-port 5011 \
  --http-port 7575 \
  > /tmp/json-api.log 2>&1 &

JSON_API_PID=$!
echo "$JSON_API_PID" > /tmp/json-api.pid

# Wait for startup
sleep 10

# Verify started
grep "Started server" /tmp/json-api.log
```

**Expected output:**
```
Started server: (ServerBinding(/127.0.0.1:7575),None)
```

**What this provides:**
- HTTP REST API for contract operations (port 7575)
- WebSocket streaming for active contracts
- Alternative to gRPC Ledger API

**Startup time:** ~10 seconds

**Time:** < 20 seconds

---

## Step 11: Verify Deployment Complete

Run all verification checks:

```bash
# Canton running
ps aux | grep canton | grep -v grep || echo "ERROR: Canton not running"

# JSON API running
ps aux | grep json-api | grep -v grep || echo "ERROR: JSON API not running"

# Ports listening
lsof -i :5011 -i :5012 -i :7575 | grep LISTEN

# Parties allocated
grep "Allocated party" /tmp/canton.log | wc -l  # Should be 3

# DAR uploaded
grep "DAR upload succeeded" /tmp/canton.log || echo "Check upload status"
```

**All checks should pass:**
- Canton process running (PID visible)
- JSON API process running (PID visible)
- Ports 5011, 5012, 7575 all LISTEN
- 3 parties allocated
- DAR upload successful

**Time:** < 1 minute

---

## Step 12: Stop Services

```bash
# Stop Canton
pkill -f canton.jar

# Stop JSON API
pkill -f json-api

# Verify stopped
ps aux | grep canton | grep -v grep || echo "Canton stopped"
ps aux | grep json-api | grep -v grep || echo "JSON API stopped"

# Clean up files (optional)
rm -f /tmp/canton-simple.conf
rm -f /tmp/canton-bootstrap.canton
rm -f /tmp/canton.log
rm -f /tmp/json-api.log
rm -f /tmp/*.pid
```

**Time:** < 30 seconds

---

## Complete Command Reference

### Installation
```bash
curl -sSL https://get.daml.com/ | sh
brew install openjdk@11
export PATH="$HOME/.daml/bin:$PATH"
export PATH="/opt/homebrew/opt/openjdk@11/bin:$PATH"
```

### Project Development
```bash
daml new <project-name>                    # Create project
daml build                                  # Build DAR file
daml test                                   # Run tests
daml script --dar <dar> --script-name Main:setup --ide-ledger
```

### Canton Deployment
```bash
# Start Canton
java -jar ~/.daml/sdk/2.10.2/canton/canton.jar daemon \
  -c /tmp/canton-simple.conf \
  --bootstrap /tmp/canton-bootstrap.canton \
  > /tmp/canton.log 2>&1 &

# Upload DAR
daml ledger upload-dar .daml/dist/<project>-0.0.1.dar \
  --host localhost --port 5011

# Start JSON API
daml json-api --ledger-host localhost \
  --ledger-port 5011 --http-port 7575 \
  > /tmp/json-api.log 2>&1 &

# Stop services
pkill -f canton.jar
pkill -f json-api
```

### Verification
```bash
ps aux | grep canton | grep -v grep        # Canton running
lsof -i :5011 -i :5012 | grep LISTEN       # Ports listening
grep "Allocated party" /tmp/canton.log     # Parties allocated
tail -20 /tmp/canton.log                   # Recent log
```

---

## Troubleshooting Quick Reference

### Java not found
**Problem:** `Unable to locate a Java Runtime`
**Solution:** Install OpenJDK 11 and configure PATH:
```bash
brew install openjdk@11
export PATH="/opt/homebrew/opt/openjdk@11/bin:$PATH"
```

### Canton won't start
**Problem:** `Protocol version is not defined for domain`
**Solution:** Add to domain config:
```hocon
init.domain-parameters.protocol-version = 5
```

### Port already in use
**Problem:** Canton fails with "Failed to bind to address"
**Solution:** Stop existing Canton and try again:
```bash
pkill -f canton.jar
# Wait 5 seconds
# Restart Canton
```

### DAR upload fails
**Problem:** `Connection refused to localhost:5011`
**Solution:** Verify Canton is running:
```bash
ps aux | grep canton | grep -v grep
lsof -i :5011 | grep LISTEN
```

### Party already exists
**Problem:** Script fails with "Party already exists"
**Solution:** This is normal! Bootstrap allocated parties. Either:
1. Restart Canton (clears memory storage)
2. Create script without party allocation (use existing parties)

---

## Time Summary

| Phase | Time |
|-------|------|
| Installation (Daml + Java) | 15-20 min |
| Environment setup | < 1 min |
| Project creation | < 5 sec |
| Contract creation | 2-3 min |
| Build and test | < 5 sec |
| Canton configuration | < 2 min |
| Canton startup | ~40 sec |
| Verification | < 30 sec |
| DAR upload | < 10 sec |
| JSON API startup | < 20 sec |
| **Total (after installation)** | **~5-7 minutes** |
| **Total (including installation)** | **~20-30 minutes** |

---

## Key Success Factors

### Must-Do Items
- Install Java 11 (not just Daml SDK)
- Configure PATH for both Daml and Java
- Include protocol version 5 in domain config
- Use `--bootstrap` flag (not config section)
- Wait 30 seconds after Canton startup
- Use `daml ledger upload-dar` (not Canton console while daemon running)

### Common Mistakes to Avoid
- Missing Java installation (build works, test fails)
- Missing protocol version (Canton fails to start)
- Using bootstrap section in config (not supported in 2.10.2)
- Running Canton console while daemon is running (port conflict)
- Not waiting for Canton startup (upload fails)

---

## Next Steps

### Development
1. Modify IOU contract (add fields, choices)
2. Create additional contracts (Asset, Token, etc.)
3. Test multi-party workflows
4. Explore advanced Daml features

### Production Deployment
1. Configure persistent storage (PostgreSQL)
2. Set up TLS for secure communication
3. Implement JWT authentication
4. Configure multiple participants
5. Set up monitoring and logging
6. Plan backup and recovery

### Integration
1. Build TypeScript client application
2. Use JSON API for web integration
3. Connect frontend to Canton
4. Implement business logic
5. Test end-to-end workflows

---

## Reference: Contract Template Structure

```daml
module Main where

import Daml.Script

-- Contract template
template TemplateName
  with
    party1 : Party      -- Field declarations
    value : Decimal
  where
    signatory party1    -- Who must authorize creation
    observer party2     -- Who can see the contract

    -- Choice definition
    choice ChoiceName : ReturnType
      with
        param : Type
      controller party1  -- Who can exercise this choice
      do
        -- Choice logic
        create this with ...  -- Or other actions

-- Test script
setup : Script ()
setup = script do
  -- Party allocation
  alice <- allocatePartyWithHint "Alice" (PartyIdHint "Alice")

  -- User creation
  aliceId <- validateUserId "alice"
  createUser (User aliceId (Some alice)) [CanActAs alice]

  -- Contract creation
  contractId <- submit alice do
    createCmd TemplateName with party1 = alice, value = 100.0

  -- Choice exercise
  submit alice do
    exerciseCmd contractId ChoiceName with param = value

  return ()
```

---

**Document version:** 1.0
**Source:** /Users/dennisonbertram/Develop/canton/documentation/LLM-GUIDES/03-DAML-DEVELOPMENT-LEARNING-LOG.md
**Created:** October 16, 2025
**For:** LLM agents and developers new to Daml
