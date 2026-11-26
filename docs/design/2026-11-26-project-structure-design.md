# BIT-Mesh: Project Structure Design

Status: **Draft, awaiting 's approval** | Date: 2026-11-26

## 1. Intent
Fix a map of BIT-Mesh before code is written, so every file has a known place, changes are planned,
and  (builds the whole codebase) stays in step with teammates who each cover their own area.

**Goals:** one core codebase that also serves as the test bench, switched by an environment variable ();
modular layout; clear module responsibilities and dependency rules; confirmed and open items
kept apart; a README that explains where things go.
**Non-goals:** writing product code; resolving open items; designing the user-facing mobile app.
**Must not happen:** a gap gets silently filled by code; a team parameter changes unnoticed; simulation
settings or results get mistaken for the real design.

## 2. Constraints
- Baseline: the two documents in `Decisions Based DOCS`. Their open items and discrepancies stay open.
- Backend code goes in `app/backend`, frontend code in `app/frontend`.
- All project documents go in `app/backend/docs` ().
- Exactly two READMEs: one in `app/backend`, one in `app/frontend` ().
- Languages: Python + Rust (confirmed by , 2026-11-26).
- The frontend is the monitoring and research dashboard only.
- No new technology or design change without team approval.

## 3. Approaches
| Approach | Benefit | Cost |
|---|---|---|
| **A. Split by running component (picked by )** | One main language per running part | Shared messages and transaction rules must be kept matching |
| B. Python leads, Rust for selected parts | Quicker early experiments | Moving boundary; needs a connecting tool; may be harder on phones |
| C. Rust leads | Node and central server share rules | Central server changes likely slower |

The language split is a new implementation choice. **Team check: not yet recorded.**

## 4. Layout
Everything below the top-level choice of A is **Claude's proposal**, approved only if this document is.

```text
app/
├── backend/
│   ├── README.md         start here: overview, map, "where does my change go?"
│   ├── docs/             all project documents, including those covering the frontend ()
│   │   ├── architecture.md    confirmed behaviour, flows, access rules, dependency rules
│   │   ├── modules.md         reference for every module, backend and frontend
│   │   ├── parameters.md      agreed values and their source
│   │   ├── open-items.md      unresolved items and discrepancies, with IDs (OPEN-001...)
│   │   ├── ownership.md       module -> teammate to consult
│   │   ├── change-process.md  how a locked item is changed
│   │   ├── decisions/         one short record per approved change
│   │   └── design/            design documents like this one
│   ├── contracts/            language-neutral message definitions
│   │   └── test-vectors/     shared inputs and expected results for Rust and Python
│   ├── config/               agreed baseline values; test-mode overrides in a separate file
│   ├── node/                 RUST: pool node
│   │   └── crates/
│   │       ├── pool/         mode switching, pool formation, role election
│   │       ├── network/      libp2p, QUIC discovery, TCP, RLPx/BIP324, Snappy + SSZ
│   │       ├── whiteboard/   IP/MAC directory, whiteboard records
│   │       ├── auth/         identity, rotating hash, 3-way hashing, HMAC, ChaCha20, nonce, keys, hardware security
│   │       ├── transaction/  transaction fields and lifecycle
│   │       ├── consensus/    validator votes, counting, threshold
│   │       ├── ledger/       account balances, chain structure when decided
│   │       ├── storage/      storage interface (engine open)
│   │       ├── sync/         reconnection and transfer to central
│   │       ├── audit/        agreed event logging
│   │       └── app/          the runnable node; connects the other crates
│   ├── central/              PYTHON: central server
│   │   ├── processing/       normal-mode transactions (flow open)
│   │   ├── sync_intake/      authenticated WebSocket, receiving pool data
│   │   ├── revalidation/     one-by-one validation as nodes connect
│   │   ├── review/           flag, re-confirm, cross-check, manual review
│   │   └── records/          central records and audit intake
│   ├── dashboard/            PYTHON: service behind the frontend
│   │   ├── monitoring/       pools, nodes, elections, votes, sync, audit views
│   │   └── controls/         central connectivity, system status, simulation controls
│   └── relay/                ESP32 + LoRa prototype (firmware language open)
└── frontend/                 dashboard UI (framework open)
    ├── README.md         frontend overview and section map
    ├── pools-nodes/
    ├── elections-voting/
    ├── transactions-sync/
    ├── security-audit/
    ├── system-status/
    └── simulation/
```
The layout shows responsibilities. Inner file layout, package names and build files are added when each
module's code starts. The WebSocket is listed because the team chose it for reconnection.

## 5. Dependency rules
1. Rust and Python never import each other's code. They interact only through messages in `contracts/`.
2. Only `node/crates/app` wires crates together. Rule crates (`transaction`, `consensus`, `ledger`,
   `pool`) do not depend on concrete networking or storage; they use interfaces.
3. The frontend talks only to the dashboard service.
4. The dashboard never edits balances, votes or transaction records directly.
5. Normal and test mode run the same core code; there is no separate test application.
6. Rules implemented in both languages must pass the same test vectors.

## 6. Test mode
There is no separate test bench module ('s decision). The team's "set up a test bench" is met by
running this same system in test mode.

**Switch (name and values are Claude's proposal):** `BITMESH_MODE=production|testbench`, read at startup by
the node, central server and dashboard service. Unset means `production`; any other value stops startup
with an error. Each running part shows its mode in its logs, and the dashboard displays it.

**Only in test mode:**
- Simulation controls in `dashboard/controls/` and `frontend/simulation/` work. Production refuses them.
- Parameter overrides are read. They live in a file separate from the agreed baseline, are visibly
  labelled, and are saved with each run's results. Production ignores that file.
- An approved temporary stand-in may run. It sits in a `standins/` folder inside the module that owns the
  feature, behind that module's interface, and is labelled in results. **No stand-ins are approved yet.**

Each module owns its own test-mode behaviour and measurements. Test mode changes conditions and approved
overrides; everything else follows the production path.

**Unresolved in either mode:** a named interface fails with a clear "undecided: OPEN-xxx" error.

**Open:**
- How several nodes are launched for a multi-node run (the variable sets the mode; it does not start nodes).
- Where run results are stored, which measurements are taken and the test cases.
- How the dashboard observes or controls nodes, including during loss of central connectivity.
- Network interruptions and reconnections are excluded from audit logging. Whether the dashboard may
  show them live, or test results may record them, is not decided.

## 7. Lock and change rules
- **Locked:** confirmed behaviour and parameters, top-level layout, module responsibilities, dependency rules.
- **Free to change:** files and code inside a module that keep to the above.
- **To change a locked item:** write a record in `backend/docs/decisions/` (what, why, who approved, date,
  affected modules), then update the docs in the same change. Team design changes need team approval.
- Parameter values in `parameters.md` and `backend/config/` must match.

## 8. Consultation (initial, from the note folders)
| Area | Consult |
|---|---|
| `network` (communication) | Farhan |
| `transaction` | Prameya |
| `relay`, hardware | Mizan |
| All other areas, including security and consensus | Unassigned |

Team notes are background. They do not override the baseline.

## 9. Module reference (`backend/docs/modules.md`)
For each module: purpose; what belongs and what does not; allowed dependencies; relevant confirmed decisions (with
source section); related open-item IDs; who to consult; current status.

## 10. What gets created on approval
The folders above, the backend and frontend READMEs, the seven documents in `backend/docs/`
(`architecture`, `modules`, `parameters`, `open-items`, `ownership`, `change-process`, and a first decision
record for the language split), and the baseline parameter file. No product code or dependencies.
Version control is not set up as part of this step.

## 11. Left to the builder
README wording, open-item numbering, and the baseline file's format (proposal: TOML, readable from
both languages). `open-items.md` must carry every open item and discrepancy from both baseline
documents, plus gaps found since: central-mode processing, account creation and opening balances,
identity (IP or key), balance updates and reservation inside one pool, vote weighting, vote timeout and
loss of a validator or whiteboard, role rounding, message formats, and the open points in section 6.
