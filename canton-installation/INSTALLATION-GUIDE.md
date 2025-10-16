# Canton Installation Guide

**Source:** https://docs.daml.com/canton/usermanual/installation.html
**Version:** Daml SDK 2.10.2
**Last Updated:** October 16, 2025

---

## Overview

This guide will guide you through the process of setting up your Canton nodes to build a distributed Daml ledger. You will learn:

- How to set up and configure a participant node
- How to set up and configure an embedded or distributed sync domain
- How to connect a participant node to a sync domain

A single Canton process can run multiple nodes, which is very useful for testing and demonstration. In a production environment, you typically run one node per process.

---

## Prerequisites

### Hardware Resources

Adequate hardware resources need to be available for each Canton node in a test, staging, or production environment. It is recommended to begin with a potentially over-provisioned system. Once a long running, performance benchmark has proven that the application's NFRs can be met (e.g., application request latency, PQS query response time, etc.) then decreasing the available resources can be tried, with a follow up rerun of the benchmark to confirm the NFRs can still be met.

**Minimum Recommended Resources:**

- **Physical Host/VM/Container:** 6 GB of RAM and at least 4 CPU cores
- **JVM Heap:** At least 4 GB RAM
- **JVM Optimization:** Add `-XX:+UseG1GC` to force the JVM to use the G1 garbage collector. Experience has shown that the JVM may use a different garbage collector in a low resource situation which can result in long latencies.

---

## Understanding Canton Topology

The Canton topology is made up of **parties**, **participants**, and **sync domains**.

### Topology Components

```
┌─────────────────────────────────────────────────────────┐
│                  Canton Network                          │
│                                                          │
│  ┌──────────────┐         ┌──────────────┐             │
│  │ Participant  │◄───────►│    Sync      │             │
│  │    Node 1    │         │   Domain     │             │
│  │              │         │              │             │
│  │ - Parties    │         │ - Sequencer  │             │
│  │ - Smart      │         │ - Mediator   │             │
│  │   Contracts  │         │ - Manager    │             │
│  └──────────────┘         └──────────────┘             │
│         ▲                        ▲                      │
│         │                        │                      │
│         │                        │                      │
│  ┌──────▼───────┐               │                      │
│  │ Participant  │───────────────┘                      │
│  │    Node 2    │                                       │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

### Key Concepts

- **Parties:** Legal entities or roles in the system (hosted on participant nodes)
- **Participant Nodes:** Run Daml code, execute smart contracts, host parties
- **Sync Domains:** Facilitate communication and synchronization between participants
  - **Sequencer:** Orders encrypted messages
  - **Mediator:** Aggregates validation responses
  - **Manager:** Verifies validity of topology changes

**Trust Model:** The Canton protocol assumes that participant nodes don't trust each other. Organizations typically run one participant node (except for scaling purposes).

**Minimum Setup:** To build a test network, you need at least one participant node and one sync domain.

---

## Step 1: Download Canton

### Open Source Edition

Download from GitHub:
- **URL:** https://github.com/digital-asset/daml/releases
- **License:** Open Source
- **Features:** Core Canton functionality

### Enterprise Edition

**For Daml Enterprise Customers:**

1. Follow the Installing Daml Enterprise instructions: https://docs.daml.com/getting-started/installation.html#installing-the-enterprise-edition
2. Download the appropriate Canton artifact
3. Alternatively, use Daml Enterprise Canton Docker images: https://docs.daml.com/canton/usermanual/docker.html

**Enterprise Features:**
- Additional sync domain drivers (Oracle, Fabric, Ethereum)
- High availability support
- Enterprise support and SLAs

---

## Step 2: Installation Directory Structure

After downloading, extract Canton to your desired location:

```bash
cd ./canton-<type>-X.Y.Z
```

### Config Directory Contents

```
.
└── config/
    ├── shared.conf          # Shared configuration (included by all)
    ├── sandbox.conf         # Simple test setup (in-memory)
    ├── participant.conf     # Participant node configuration
    ├── domain.conf          # Embedded sync domain configuration
    ├── sequencer.conf       # Sequencer node (Enterprise)
    ├── mediator.conf        # Mediator node (Enterprise)
    ├── manager.conf         # Manager node (Enterprise)
    ├── jwt/                 # JWT configuration for Ledger API
    ├── misc/                # Miscellaneous configs
    ├── monitoring/          # Metrics and tracing configs
    ├── remote/              # Remote console connection configs
    ├── storage/             # Storage layer configurations
    │   ├── postgres.conf
    │   ├── oracle.conf
    │   └── memory.conf
    ├── tls/                 # TLS certificates and configs
    │   └── gen-test-certs.sh
    └── utils/               # Utility scripts (e.g., database setup)
```

### Configuration Files

**Node Configurations:**
- `participant.conf` - Participant node configuration
- `domain.conf` - Embedded sync domain (open source option)
- `sequencer.conf`, `mediator.conf`, `manager.conf` - Distributed sync domain (Enterprise)
- `sandbox.conf` - Simple test setup with in-memory storage

**Shared Configuration:**
- `shared.conf` - Included by all node configurations

---

## Step 3: Select Your Storage Layer

Canton supports multiple storage backends:

### Option 1: In-Memory (Development/Testing)

- **Pros:** Simple, no setup required, fast
- **Cons:** Data not persisted, lost on restart
- **Use Case:** Development, demos, testing

**Configuration:** Works out of the box with `sandbox.conf`

### Option 2: PostgreSQL (Recommended for Production)

- **Pros:** Production-ready, tested, reliable
- **Cons:** Requires database setup
- **Use Case:** Production deployments

**Supported Versions:** Canton is tested on currently supported PostgreSQL versions (see release notes for specific versions)

### Option 3: Oracle (Enterprise Only)

- **Pros:** Enterprise database features
- **Cons:** Requires license, more complex
- **Use Case:** Enterprise environments with existing Oracle infrastructure

---

## Step 4: Set Up PostgreSQL (Production)

### Install PostgreSQL

Make sure you have a running PostgreSQL server. Canton is tested on currently supported Postgres versions: https://www.postgresql.org/support/versioning/

### Create Databases and User

Canton provides a helper script for database setup.

#### Using the Provided Script (with Docker)

```bash
cd ./config/utils/postgres
./db.sh start [durable]
./db.sh setup
```

**Script Details:**
- Reads connection settings from `config/utils/postgres/db.env`
- Starts a Docker-based Postgres instance (optional)
- Creates databases defined in `config/utils/postgres/databases`
- Use `start durable` to persist data between restarts

**Additional Commands:**
- `create-user` - Create the database user
- `resume` - Resume a stopped Docker Postgres instance
- `drop` - Drop created databases

#### Manual Database Setup

If you have an existing PostgreSQL server:

1. Create one database per Canton node
2. Create a database user with appropriate permissions
3. Note the connection details for configuration

### Configure Database Connectivity

**Option 1: Environment Variables**

```bash
export POSTGRES_HOST="localhost"
export POSTGRES_USER="test-user"
export POSTGRES_PASSWORD="test-password"
export POSTGRES_DB="postgres"
export POSTGRES_PORT=5432
```

**Option 2: Edit Configuration File**

Edit `config/storage/postgres.conf` with your database settings.

### Storage Mixin Configuration

The storage configuration uses a "mixin" pattern. By default, `config/shared.conf` includes `postgres.conf`.

**In node configurations, reference shared storage:**

```hocon
storage = ${_shared.storage}
storage.parameters.databaseName = "canton_participant"
```

**Common Error:** If you see `Could not resolve substitution to a value: ${_shared.storage}`, you forgot to include the persistence mixin configuration file.

### Database Performance Tuning

Canton requires a properly sized and correctly configured database. See the **Postgres performance guide** for:
- Connection pool sizing
- Query optimization
- Index management
- Performance tuning parameters

**Important Note:** Multiple versions of Postgres are tested, including serverless variants. However, serverless Postgres has transient behaviors requiring robust application testing to verify SLAs are met.

---

## Step 5: Generate TLS Certificates

The reference configurations use TLS to secure APIs.

### TLS Configuration Files

Located in `config/tls/`:

- **`tls-ledger-api.conf`** - TLS for Ledger API (participant)
- **`mtls-admin-api.conf`** - mTLS for Administration API (all nodes)
- **`tls-public-api.conf`** - TLS for Public API (sequencer/sync domain)

### Generate Test Certificates

For development/testing, use the provided script:

```bash
cd config/tls
./gen-test-certs.sh
```

This generates self-signed certificates with correct SAN entries.

### Production Certificates

For production:
1. Obtain certificates from a trusted Certificate Authority
2. Ensure certificates include proper SAN entries for node addresses
3. Configure certificate paths in TLS configuration files

### Authentication Methods

- **Public API:** Built-in client authentication using node identity keys (cannot be disabled)
- **Ledger API:** JWT-based authentication (see `config/jwt/`)
- **Admin API:** Optional mTLS authentication

### Skip TLS (Development Only)

For quick testing, you can comment out TLS includes in configuration files. **NOT recommended for production.**

---

## Step 6: Set Up a Participant Node

### Start the Participant

Using the reference configuration:

```bash
./bin/canton [daemon] -c config/participant.conf
```

**Arguments:**
- `daemon` (optional) - Run as daemon without interactive console
- `-c` - Specify configuration file(s)

**Additional Options:**
- Logging configuration tuning
- JVM options
- Multiple configuration files

### Automatic Initialization

By default, the node initializes itself automatically:
- Creates necessary keys
- Generates topology transactions
- Initializes using the name from the configuration file

See **Identity Management** documentation for manual initialization.

### Secure the APIs

#### 1. Network Binding

By default, APIs bind to `localhost` only. To accept remote connections:

```hocon
canton.participants.participant1.ledger-api.address = "0.0.0.0"
canton.participants.participant1.admin-api.address = "0.0.0.0"
```

Or bind to a specific interface/hostname.

#### 2. Administration API Security

**Recommended:** Set up mutual TLS authentication

- Configure in `config/tls/mtls-admin-api.conf`
- Require client certificates
- Restrict access to authorized administrators

#### 3. Ledger API Security

**Recommended Security Measures:**

1. **Enable TLS**
   - Configure in `config/tls/tls-ledger-api.conf`
   - Use production certificates

2. **JWT Authentication**
   - Configure JWT token validation
   - See examples in `config/jwt/`
   - Implement proper token generation and validation

### Configure Applications and Users

Canton distinguishes between:

**Static Configuration:**
- Items in configuration files
- Port bindings
- Storage settings
- Network addresses

**Dynamic Configuration:**
- Daml archives (DARs)
- Sync domain connections
- Party hosting
- Runtime topology changes

**Dynamic changes are made through:**
- Console commands
- Administration APIs

### Next Steps for Participants

1. Connect to sync domains
2. Onboard parties
3. Deploy Daml applications (DARs)
4. Configure user access

See the **Getting Started Guide** for details.

---

## Step 7: Set Up a Synchronization Domain

A sync domain consists of three processes:
- **Sequencer:** Orders encrypted messages
- **Mediator:** Aggregates validation responses
- **Manager:** Verifies topology changes

These nodes don't store ledger data; they facilitate participant communication.

### Choose a Domain Driver

**Available Drivers:**

1. **Postgres-based** (Open Source & Enterprise)
   - Database-backed synchronization
   - Crash recovery support
   - Simple setup

2. **Oracle-based** (Enterprise Only)
   - Enterprise database features
   - See Oracle domain documentation

3. **Hyperledger Fabric-based** (Enterprise Only)
   - Blockchain-backed domain
   - BFT consensus

4. **Ethereum-based** (Enterprise Only)
   - Public/private blockchain integration
   - Smart contract sequencing

5. **Native Canton BFT** (Coming Soon)
   - Used for Canton Network
   - Byzantine fault tolerance

---

## Option A: Embedded Sync Domain (Simplest)

The embedded sync domain runs all three processes (sequencer, mediator, manager) in a single node.

### Start Embedded Domain

```bash
./bin/canton -c config/domain.conf
```

### Characteristics

- **Pros:** Simple setup, single process
- **Cons:** No high availability (Enterprise required for HA)
- **Use Case:** Development, testing, open source deployments
- **Crash Recovery:** Supported
- **High Availability:** Not supported (use microservices approach)

---

## Option B: Microservices Sync Domain (Enterprise)

Run sync domain processes as separate microservices for high availability.

### Start Individual Services

```bash
# Start each service separately
./bin/canton daemon -c config/sequencer.conf
./bin/canton daemon -c config/mediator.conf
./bin/canton daemon -c config/manager.conf
```

### Initialize and Connect Services

#### Option 1: Remote Console

```bash
./bin/canton -c config/remote/mediator.conf,config/remote/manager.conf,config/remote/sequencer.conf
```

Then run bootstrap command:

```scala
manager.setup.bootstrap_domain(sequencers.all, mediators.all)
```

#### Option 2: Programmatic Bootstrap

See the detailed guide on **domain bootstrapping** for programmatic initialization.

### Benefits of Microservices

- High availability configuration
- Independent scaling
- Fault isolation
- Rolling updates
- Enterprise support

---

## Step 8: Connect Participant to Sync Domain

### Connection Command

From the participant's console:

```scala
participant.domains.connect(
  "domain",  // Domain alias
  "https://localhost:10018",  // Sequencer URL
  certificatesPath = "config/tls/root-ca.crt"  // CA certificate
)
```

**Parameters:**
- `domain` - Local alias for the domain
- URL - Sequencer public API endpoint
- `certificatesPath` - CA certificate (needed for self-signed certs)

### Verify Connection

```scala
participant.domains.list_connected()
```

Should show the connected domain.

### Secure the Domain APIs

1. **Admin API:** Use mTLS (mutual TLS)
2. **Public API:** TLS is mandatory, cannot be disabled
3. **Network Binding:** Configure appropriate interface (0.0.0.0 or specific IP)

### Next Steps for Domains

1. **Control Access:** Configure domain to be permissioned (default is open)
2. **High Availability:** Set up HA for production
3. **Monitoring:** Implement health checks and metrics
4. **Backup:** Configure regular backups

---

## Multi-Node Setup (Testing/Demo)

For testing, you can run multiple nodes in a single process.

### Method 1: Multiple Configuration Files

```bash
./bin/canton \
  -c config/participant.conf \
  -c config/sequencer.conf,config/mediator.conf \
  -c config/manager.conf
```

### Method 2: Combined Configuration File

Create a single configuration file with multiple node definitions.

### Use Cases

- **Development:** Test multi-party workflows locally
- **Demos:** Show distributed ledger without complex setup
- **Testing:** Integration tests with multiple nodes

**Not Recommended for Production:** Use separate processes per node in production.

---

## Configuration Reference

### Static vs Dynamic Configuration

**Static Configuration (config files):**
- Port bindings
- Storage settings
- TLS certificates
- Network addresses
- JVM settings

**Dynamic Configuration (console/API):**
- DAR deployments
- Domain connections
- Party onboarding
- Topology changes
- Runtime settings

### Configuration Mixins

Mixins allow configuration reuse:

```hocon
# In shared.conf
_shared {
  storage = ${postgres.storage}
}

# In node config
storage = ${_shared.storage}
storage.parameters.databaseName = "my_database"
```

**Available Mixins:**
- Storage configurations (`storage/`)
- TLS configurations (`tls/`)
- JWT configurations (`jwt/`)
- Monitoring configurations (`monitoring/`)

---

## Deployment Models

### Development/Testing

- In-memory storage
- Single process, multiple nodes
- Self-signed certificates
- Minimal security

### Staging

- PostgreSQL storage
- Separate processes per node
- Proper TLS certificates
- Basic authentication
- Monitoring enabled

### Production

- PostgreSQL/Oracle storage with proper tuning
- One node per process, one process per host/container
- Production TLS certificates from CA
- Full authentication (mTLS + JWT)
- High availability configuration
- Disaster recovery setup
- Comprehensive monitoring and alerting
- Regular backups
- Security hardening

---

## High Availability & Disaster Recovery

### Participant HA

- Active-active configuration
- Shared database
- Load balancing
- See: **HA Participant Architecture**

### Domain HA

- Multiple sequencer replicas
- Multiple mediator replicas
- Database replication
- See: **Components for HA**

### Disaster Recovery

- Regular database backups
- Configuration backups
- Key material backup
- Recovery procedures
- See: **Disaster Recovery Guide**

---

## Monitoring and Operations

### Health Checks

- API health endpoints
- Database connectivity
- Domain connectivity
- Resource utilization

### Metrics

- Transaction throughput
- Ledger API latency
- Database performance
- JVM metrics

### Logging

- Configure log levels
- Structured logging
- Log aggregation
- Audit trails

### Configuration Files

See `config/monitoring/` for:
- Prometheus integration
- Tracing configuration
- Metrics export

---

## Security Best Practices

### Network Security

1. **Firewall Rules:** Restrict access to required ports only
2. **TLS Everywhere:** All API communication over TLS
3. **Network Segmentation:** Isolate Canton nodes
4. **DDoS Protection:** Implement rate limiting

### Authentication & Authorization

1. **mTLS for Admin API:** Require client certificates
2. **JWT for Ledger API:** Implement token-based auth
3. **Strong Passwords:** For database access
4. **Principle of Least Privilege:** Grant minimal required permissions

### Key Management

1. **Secure Key Storage:** Protect private keys
2. **Key Rotation:** Regular certificate rotation
3. **Backup Keys:** Secure backup of identity keys
4. **Access Control:** Restrict key access

### Database Security

1. **Encrypted Connections:** SSL/TLS to database
2. **Strong Passwords:** Database user passwords
3. **Access Control:** Database firewall rules
4. **Encryption at Rest:** Enable database encryption

---

## Troubleshooting

### Common Issues

**"Could not resolve substitution to a value: ${_shared.storage}"**
- Missing storage mixin configuration
- Solution: Include storage config file

**Connection Refused**
- API not bound to correct interface
- Solution: Check `address` in config

**Certificate Errors**
- Self-signed certificates not trusted
- Solution: Use `certificatesPath` parameter or install CA cert

**Database Connection Errors**
- Incorrect credentials or connectivity
- Solution: Verify environment variables and network access

### Getting Help

- **Documentation:** https://docs.daml.com/canton/
- **Community Forum:** http://discuss.daml.com/
- **GitHub Issues:** https://github.com/digital-asset/daml/issues
- **Enterprise Support:** https://support.digitalasset.com/s/

---

## Next Steps

After installation:

1. **Complete Getting Started Tutorial**
   - https://docs.daml.com/canton/tutorials/getting_started.html

2. **Deploy Daml Applications**
   - Upload DARs
   - Onboard parties
   - Test workflows

3. **Configure High Availability**
   - Set up HA for production
   - Test failover procedures

4. **Implement Monitoring**
   - Set up metrics collection
   - Configure alerting
   - Create dashboards

5. **Security Hardening**
   - Production certificates
   - Full authentication
   - Security audit

6. **Performance Tuning**
   - Database optimization
   - JVM tuning
   - Load testing

---

## Additional Resources

### Documentation

- **Main Documentation:** https://docs.daml.com/
- **Canton User Manual:** https://docs.daml.com/canton/usermanual/
- **API Reference:** https://docs.daml.com/canton/reference/
- **Configuration Reference:** https://docs.daml.com/canton/usermanual/static_conf.html

### Downloads

- **Open Source Releases:** https://github.com/digital-asset/daml/releases
- **Enterprise Downloads:** https://docs.daml.com/getting-started/installation.html

### Community

- **Forum:** http://discuss.daml.com/
- **GitHub:** https://github.com/digital-asset/daml
- **Blog:** https://blog.digitalasset.com/

### Support

- **Community Support:** discuss.daml.com
- **Enterprise Support:** support.digitalasset.com

---

## Appendix: Quick Reference

### Essential Commands

```bash
# Start participant with console
./bin/canton -c config/participant.conf

# Start participant as daemon
./bin/canton daemon -c config/participant.conf

# Start domain
./bin/canton -c config/domain.conf

# Multiple config files
./bin/canton -c config1.conf,config2.conf

# Remote console
./bin/canton -c config/remote/participant.conf
```

### Console Commands

```scala
// List participants
participants.list

// Connect to domain
participant.domains.connect("domain", "https://localhost:10018")

// List parties
participant.parties.list()

// Upload DAR
participant.dars.upload("path/to/file.dar")

// Domain bootstrap (Enterprise)
manager.setup.bootstrap_domain(sequencers.all, mediators.all)
```

### Directory Structure

```
canton-X.Y.Z/
├── bin/
│   └── canton              # Main executable
├── config/                 # Configuration files
├── lib/                    # JAR files
└── examples/               # Example applications
```

---

**Copyright © 2025 Digital Asset (Switzerland) GmbH and/or its affiliates. All rights reserved.**
