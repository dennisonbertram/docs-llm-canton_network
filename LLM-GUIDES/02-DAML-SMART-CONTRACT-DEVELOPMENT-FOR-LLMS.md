# Daml Smart Contract Development Guide for LLMs

**Purpose:** Rapid Daml application development for LLM-assisted coding
**Target:** AI assistants helping developers build Canton/Daml applications
**Last Updated:** October 16, 2025

---

## Quick Context for LLMs

Daml is a smart contract language for multi-party workflows with built-in privacy. When assisting with Daml development:
- Daml is **functional** and **strongly typed** (similar to Haskell)
- Contracts define **who can do what** with explicit authorization
- All state changes require **consent from authorized parties**
- Privacy is built-in: only parties to a contract see it
- Focus on **business logic**, not blockchain internals

---

## Development Workflow Checklist

```
□ Install Daml SDK
□ Create new Daml project
□ Write contract templates
□ Test contracts in Daml Studio
□ Build project (generate DAR)
□ Deploy to Canton
□ Test via Ledger API
□ Build UI application (optional)
```

---

## Quick Start: First Daml Contract in 10 Minutes

### Step 1: Install Daml SDK (2 minutes)

```bash
# macOS/Linux
curl -sSL https://get.daml.com/ | sh

# Verify installation
daml version
```

**What this installs:**
- Daml compiler
- Daml Studio (VS Code extension)
- Local sandbox for testing
- Code generation tools

### Step 2: Create Project (1 minute)

```bash
# Create new project from template
daml new my-app

# Navigate to project
cd my-app

# Project structure created:
# my-app/
# ├── daml/              # Smart contracts go here
# │   └── Main.daml      # Example contract
# ├── daml.yaml          # Project configuration
# └── package.json       # JavaScript dependencies (for UI)
```

### Step 3: Write Your First Contract (5 minutes)

Edit `daml/Main.daml`:

```daml
module Main where

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
      controller owner  -- Only owner can transfer
      do
        create this with owner = newOwner

    -- Issuer can settle by paying
    choice Settle : ()
      controller issuer  -- Only issuer can settle
      do
        return ()  -- Archive the contract
```

**Key Daml concepts shown:**
- `template` - Defines a contract type
- `with` - Contract fields/data
- `signatory` - Who must authorize creation
- `observer` - Who can see the contract
- `choice` - Actions that can be taken
- `controller` - Who can exercise the choice

### Step 4: Test in Daml Studio (2 minutes)

```bash
# Open in VS Code with Daml extension
daml studio
```

**In Daml Studio:**
1. File opens with syntax highlighting
2. Press **Cmd/Ctrl + Shift + P** → "Daml: Start Sandbox"
3. Sandbox starts with your contracts loaded
4. Test scenarios in the Navigator UI

---

## Daml Language Fundamentals for LLMs

### Contract Structure Template

```daml
module MyModule where

template ContractName
  with
    field1 : Type1
    field2 : Type2
    party : Party
  where
    signatory party              -- Required: who authorizes

    observer [otherParty]        -- Optional: who can see

    ensure someCondition         -- Optional: validity check

    choice ActionName : ReturnType
      with
        choiceParam : ParamType
      controller controllingParty
      do
        -- Action logic here
        create newContract
```

### Common Types

```daml
-- Basic types
Text           -- "hello"
Int            -- 42
Decimal        -- 3.14
Bool           -- True / False
Party          -- Ledger identity
Date           -- 2025-01-01
Time           -- Timestamps

-- Collection types
[Int]          -- List of integers
Optional Text  -- Maybe has a value
(Int, Text)    -- Tuple

-- Custom types
data Status = Pending | Approved | Rejected
```

### Key Keywords

| Keyword | Purpose | Example |
|---------|---------|---------|
| `template` | Define contract type | `template Iou with...` |
| `signatory` | Who must authorize | `signatory issuer` |
| `observer` | Who can see | `observer owner` |
| `choice` | Action on contract | `choice Transfer : ...` |
| `controller` | Who can act | `controller owner` |
| `do` | Action block | `do create this` |
| `ensure` | Validation rule | `ensure amount > 0.0` |
| `create` | Create new contract | `create Iou with ...` |
| `exercise` | Execute choice | `exercise iouId Transfer with ...` |

---

## Common Contract Patterns

### Pattern 1: Simple Asset

```daml
template Token
  with
    issuer : Party
    owner : Party
    amount : Decimal
  where
    signatory issuer, owner

    choice Split : (ContractId Token, ContractId Token)
      with
        splitAmount : Decimal
      controller owner
      do
        assert (splitAmount > 0.0 && splitAmount < amount)
        token1 <- create this with amount = splitAmount
        token2 <- create this with amount = amount - splitAmount
        return (token1, token2)

    choice Merge : ContractId Token
      with
        otherTokenId : ContractId Token
      controller owner
      do
        otherToken <- fetch otherTokenId
        assert (otherToken.issuer == issuer)
        assert (otherToken.owner == owner)
        archive otherTokenId
        create this with amount = amount + otherToken.amount
```

### Pattern 2: Proposal/Accept Pattern

```daml
-- Step 1: Propose
template TransferProposal
  with
    asset : ContractId Asset
    from : Party
    to : Party
  where
    signatory from
    observer to

    choice Accept : ContractId Asset
      controller to
      do
        exercise asset Transfer with newOwner = to

    choice Reject : ()
      controller to
      do
        return ()

-- Step 2: Create proposal
createTransferProposal assetId fromParty toParty = do
  create TransferProposal with
    asset = assetId
    from = fromParty
    to = toParty
```

### Pattern 3: Multi-Party Workflow

```daml
template TradeAgreement
  with
    buyer : Party
    seller : Party
    asset : Text
    price : Decimal
    buyerApproved : Bool
    sellerApproved : Bool
  where
    signatory buyer, seller
    ensure price > 0.0

    choice BuyerApprove : ContractId TradeAgreement
      controller buyer
      do
        create this with buyerApproved = True

    choice SellerApprove : ContractId TradeAgreement
      controller seller
      do
        create this with sellerApproved = True

    choice Settle : ()
      controller buyer, seller
      do
        assert (buyerApproved && sellerApproved)
        -- Transfer asset and payment
        return ()
```

### Pattern 4: Time-Based Contract

```daml
template Escrow
  with
    payer : Party
    payee : Party
    amount : Decimal
    releaseTime : Time
  where
    signatory payer
    observer payee

    choice Release : ()
      controller payer
      do
        currentTime <- getTime
        assert (currentTime >= releaseTime)
        -- Transfer payment
        return ()

    choice Cancel : ()
      controller payer
      do
        currentTime <- getTime
        assert (currentTime < releaseTime)
        return ()
```

---

## Building and Testing

### Compile Contract

```bash
# Build the project (creates .dar file)
daml build

# Output: .daml/dist/my-app-0.1.0.dar
```

**DAR file** = Daml Archive (compiled contracts + metadata)

### Run Tests

Create `daml/Test.daml`:

```daml
module Test where

import Main
import Daml.Script

testIouTransfer : Script ()
testIouTransfer = do
  -- Allocate parties
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"
  charlie <- allocateParty "Charlie"

  -- Alice creates IOU to Bob
  iouId <- submit alice do
    createCmd Iou with
      issuer = alice
      owner = bob
      amount = 100.0

  -- Bob transfers to Charlie
  newIouId <- submit bob do
    exerciseCmd iouId Transfer with newOwner = charlie

  -- Verify Charlie owns it
  Some iou <- queryContractId charlie newIouId
  assert (iou.owner == charlie)

  return ()
```

Run tests:

```bash
daml test
```

---

## Deploying to Canton

### Step 1: Start Canton

```bash
./bin/canton -c config/sandbox.conf
```

### Step 2: Upload DAR

```bash
# In Canton console:
participant1.dars.upload(".daml/dist/my-app-0.1.0.dar")
```

Or from command line:

```bash
daml ledger upload-dar .daml/dist/my-app-0.1.0.dar \
  --host localhost --port 5011
```

### Step 3: Allocate Parties

```scala
// In Canton console:
val alice = participant1.parties.enable("Alice")
val bob = participant1.parties.enable("Bob")
```

### Step 4: Test via Ledger API

```bash
# Create a contract
daml ledger create Iou \
  --host localhost --port 5011 \
  --application-id my-app \
  --party Alice \
  --arg '{"issuer":"Alice","owner":"Bob","amount":"100.0"}'

# Query active contracts
daml ledger fetch-dar \
  --host localhost --port 5011 \
  --party Bob
```

---

## Interacting with Contracts

### Via Daml Script

```daml
import Daml.Script

setupScenario : Script ()
setupScenario = do
  -- Create parties
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"

  -- Create IOU
  iouId <- submit alice do
    createCmd Iou with
      issuer = alice
      owner = bob
      amount = 100.0

  -- Exercise choice
  submit bob do
    exerciseCmd iouId Transfer with newOwner = alice

  return ()
```

Run script:

```bash
daml script \
  --dar .daml/dist/my-app-0.1.0.dar \
  --script-name Main:setupScenario \
  --ledger-host localhost \
  --ledger-port 5011
```

### Via HTTP JSON API

```bash
# Start JSON API server
daml json-api \
  --ledger-host localhost \
  --ledger-port 5011 \
  --http-port 7575 \
  --application-id my-app

# Create contract (HTTP POST)
curl -X POST http://localhost:7575/v1/create \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "templateId": "Main:Iou",
    "payload": {
      "issuer": "Alice",
      "owner": "Bob",
      "amount": "100.0"
    }
  }'
```

---

## Common Daml Patterns for LLMs

### Pattern: Conditional Logic

```daml
choice ApproveOrReject : Either Text (ContractId Approved)
  with
    decision : Bool
    reason : Text
  controller reviewer
  do
    if decision
      then do
        approved <- create Approved with ...
        return (Right approved)
      else
        return (Left reason)
```

### Pattern: Querying Contracts

```daml
findActiveContracts : Party -> Script [ContractId Iou]
findActiveContracts party = do
  ious <- query @Iou party
  return (map fst ious)  -- Extract contract IDs
```

### Pattern: Error Handling

```daml
choice SafeTransfer : Either Text (ContractId Token)
  with
    newOwner : Party
  controller owner
  do
    if newOwner == owner
      then return (Left "Cannot transfer to self")
      else do
        newToken <- create this with owner = newOwner
        return (Right newToken)
```

### Pattern: Assertions

```daml
template Invoice
  with
    amount : Decimal
    dueDate : Date
  where
    signatory issuer

    -- Validate at creation time
    ensure amount > 0.0
    ensure dueDate > date 2025 Jan 1

    choice Pay : ()
      controller payer
      do
        currentDate <- getTime
        -- Runtime assertion
        assert (toDateUTC currentDate <= dueDate)
        return ()
```

---

## Integration with External Systems

### JavaScript/TypeScript Client

```bash
# Generate TypeScript types from Daml
daml codegen js .daml/dist/my-app-0.1.0.dar -o ui/daml.js
```

```typescript
import { Iou } from './daml.js/my-app/lib/Main';
import Ledger from '@daml/ledger';

const ledger = new Ledger({
  token: 'your-jwt-token',
  httpBaseUrl: 'http://localhost:7575'
});

// Create IOU
const iou = await ledger.create(Iou, {
  issuer: 'Alice',
  owner: 'Bob',
  amount: '100.0'
});

// Exercise choice
const result = await ledger.exercise(
  Iou.Transfer,
  iou.contractId,
  { newOwner: 'Charlie' }
);
```

### Python Client

```python
from dazl import Network

network = Network()
network.set_config(url='http://localhost:7575')

@network.party('Alice')
async def main(client):
    # Create IOU
    iou = await client.submit_create(
        'Main:Iou',
        {
            'issuer': 'Alice',
            'owner': 'Bob',
            'amount': '100.0'
        }
    )

    # Query contracts
    contracts = await client.query('Main:Iou')
    print(f"Found {len(contracts)} IOUs")

network.run_forever()
```

---

## Debugging Tips for LLMs

### Common Errors

**Error:** `Variable not in scope`
```daml
-- Wrong:
choice Transfer
  do
    create Iou with amount = newAmount  -- newAmount not defined

-- Right:
choice Transfer
  with
    newAmount : Decimal  -- Define parameter
  do
    create Iou with amount = newAmount
```

**Error:** `Couldn't match expected type`
```daml
-- Wrong:
amount : Int
-- ...
create this with amount = 100.0  -- 100.0 is Decimal, not Int

-- Right:
amount : Decimal
create this with amount = 100.0
```

**Error:** `Missing authorization`
```daml
-- Wrong:
template Contract
  with
    party1 : Party
  where
    observer party1  -- observer can't authorize creation

-- Right:
template Contract
  with
    party1 : Party
  where
    signatory party1  -- signatory can authorize
```

### Debugging Commands

```bash
# Check syntax errors
daml build

# Run tests with verbose output
daml test --color

# View compiled code
daml damlc inspect-dar .daml/dist/my-app-0.1.0.dar

# Start sandbox with logging
daml sandbox --log-level DEBUG
```

---

## Project Structure Best Practices

```
my-app/
├── daml/
│   ├── Main.daml           # Main contracts
│   ├── Types.daml          # Shared data types
│   ├── Workflows.daml      # Business logic
│   ├── Setup.daml          # Setup scripts
│   └── Test/
│       ├── Main.daml       # Unit tests
│       └── Integration.daml # Integration tests
├── daml.yaml               # Project config
├── ui/                     # Frontend (optional)
│   ├── src/
│   └── package.json
└── README.md
```

### daml.yaml Configuration

```yaml
sdk-version: 2.10.2
name: my-app
version: 0.1.0
source: daml
init-script: Setup:initialize
parties:
  - Alice
  - Bob
dependencies:
  - daml-prim
  - daml-stdlib
build-options:
  - --enable-scenarios
```

---

## Testing Strategies

### Unit Tests (Daml Script)

```daml
-- Test individual choices
testTransfer : Script ()
testTransfer = do
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"

  iouId <- submit alice do
    createCmd Iou with
      issuer = alice
      owner = bob
      amount = 100.0

  -- Test transfer
  newId <- submit bob do
    exerciseCmd iouId Transfer with newOwner = alice

  -- Verify result
  Some iou <- queryContractId alice newId
  assert (iou.owner == alice)
```

### Integration Tests (Daml Script)

```daml
-- Test full workflow
testFullWorkflow : Script ()
testFullWorkflow = do
  -- Setup parties
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"
  charlie <- allocateParty "Charlie"

  -- Step 1: Create IOU
  iouId <- submit alice do
    createCmd Iou with ...

  -- Step 2: Transfer to Bob
  iouId2 <- submit bob do
    exerciseCmd iouId Transfer with ...

  -- Step 3: Settle
  submit alice do
    exerciseCmd iouId2 Settle

  -- Verify final state
  active <- queryContractId alice iouId2
  assert (isNone active)  -- Contract archived
```

### Property-Based Tests

```daml
-- Test invariants
testAmountPositive : Script ()
testAmountPositive = do
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"

  -- This should fail due to ensure clause
  submitMustFail alice do
    createCmd Iou with
      issuer = alice
      owner = bob
      amount = -100.0  -- Negative amount
```

---

## LLM Response Templates

### When user asks: "How do I create a Daml contract?"

```
Here's a simple Daml contract template:

```daml
module Main where

template MyContract
  with
    party1 : Party
    party2 : Party
    data : Text
  where
    signatory party1
    observer party2

    choice DoSomething : ()
      controller party1
      do
        return ()
```

Key components:
- `template` - Defines the contract type
- `with` - Contract data fields
- `signatory` - Who must authorize creation
- `observer` - Who can see the contract
- `choice` - Actions that can be performed

To test it:
1. Save as `daml/Main.daml`
2. Run `daml build`
3. Run `daml test`
```

### When user asks: "How do I deploy to Canton?"

```
To deploy your Daml app to Canton:

1. Build the DAR:
   ```bash
   daml build
   ```

2. Start Canton:
   ```bash
   ./bin/canton -c config/sandbox.conf
   ```

3. Upload in Canton console:
   ```scala
   participant1.dars.upload(".daml/dist/my-app-0.1.0.dar")
   ```

4. Create parties:
   ```scala
   val alice = participant1.parties.enable("Alice")
   val bob = participant1.parties.enable("Bob")
   ```

5. Test it works:
   ```scala
   participant1.dars.list()  // Should show your DAR
   participant1.parties.list()  // Should show Alice and Bob
   ```
```

### When user needs help with contract logic:

```
For [describe use case], here's a pattern:

```daml
template [ContractName]
  with
    [fields]
  where
    signatory [who authorizes]
    observer [who can see]

    choice [ActionName] : [ReturnType]
      with
        [parameters]
      controller [who can act]
      do
        [business logic]
        create/exercise/return
```

Specific to your case:
- Use `signatory` for parties that must authorize
- Use `observer` for parties that need visibility
- Use `ensure` for validation rules
- Use `assert` for runtime checks
```

---

## Quick Reference: Daml Commands

```bash
# PROJECT MANAGEMENT
daml new my-app              # Create new project
daml build                   # Compile contracts
daml test                    # Run tests
daml clean                   # Clean build artifacts

# DEVELOPMENT
daml studio                  # Open in VS Code
daml sandbox                 # Start local ledger
daml navigator              # Open UI for testing

# CODE GENERATION
daml codegen js <dar> -o <output>   # Generate TypeScript
daml codegen java <dar> -o <output> # Generate Java

# DEPLOYMENT
daml ledger upload-dar <dar>        # Upload to ledger
daml ledger list-parties            # List parties
daml script --dar <dar> --script-name <name>  # Run script

# UTILITIES
daml version                 # Show version
daml install <version>       # Install SDK version
daml damlc inspect-dar <dar> # Inspect DAR contents
```

---

## Summary for LLMs

When helping users build Daml applications:

1. **Start simple:** Basic template → Add choices → Add logic
2. **Test early:** Use Daml Script for testing before deploying
3. **Follow patterns:** Proposal/accept, multi-party, time-based
4. **Think authorization:** Every action needs a controller
5. **Privacy by design:** Only signatories/observers see contracts
6. **Type safety:** Leverage strong typing to catch errors early

**Typical workflow:**
```
Write contract → Test in Studio → Build DAR → Deploy to Canton → Test via API
```

**Most common tasks:**
- Creating templates: Define `with` fields, `signatory`, and `choice`
- Testing: Use Daml Script with `submit`, `create`, `exercise`
- Deploying: `daml build` + upload DAR to Canton
- Integration: Generate code + use Ledger API

**Remember:** Daml is about **who can do what**, not **how blockchain works**. Focus on business logic and authorization.
