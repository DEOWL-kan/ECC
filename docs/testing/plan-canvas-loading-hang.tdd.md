# Plan Canvas loading hang TDD evidence

## Source

No implementation plan was supplied. The journeys and guarantees below were
derived from the reported intermittent localhost loading hang, especially when
opening a second Plan Canvas in one agent chat.

## User journeys

1. As a reviewer opening plans from multiple ECC worktrees, I want every new
   Canvas to use the current server code so an older same-version process cannot
   disable or stall the page.
2. As a reviewer opening a plan with Mermaid diagrams, I want the Canvas page to
   finish loading even when the Mermaid CDN is slow or unavailable.

## Task report

### Replace a stale same-version detached server

- RED: `node tests/integration/plan-canvas-e2e.test.js` produced 9 passes and
  1 failure. The current CLI reused a fake legacy server that reported the same
  package version and sent the session-open request to it.
- RED checkpoint: `5dc6d85c test: add reproducer for stale plan canvas server`.
- GREEN: the health handshake now carries a protocol version and a SHA-256
  fingerprint of every module loaded into the detached Canvas server. The CLI
  retires any process whose package, protocol, or runtime fingerprint differs.
- GREEN: `node tests/integration/plan-canvas-e2e.test.js` produced 10 passes and
  0 failures.
- GREEN checkpoint: `7e0d115e fix: restart stale plan canvas servers`.

### Keep Mermaid enhancement from blocking page load

- RED: `node tests/scripts/plan-canvas.test.js` produced 27 passes and 1
  failure. The generated artifact loaded Mermaid with top-level `await`, which
  allowed an unresolved remote import to hold the document load event open.
- RED checkpoint: `ca0d0415 test: reproduce mermaid page load stall`.
- GREEN: the dynamic Mermaid import now starts from an async `load` listener.
  Browsers do not await that listener, so diagrams remain progressive
  enhancement and the raw Mermaid source remains available during a network
  stall.
- GREEN: `node tests/scripts/plan-canvas.test.js` produced 28 passes and 0
  failures.
- GREEN checkpoint: `7fc31254 fix: keep mermaid from blocking canvas load`.

## Test specification

| # | What is guaranteed | Test target | Type | Result |
| --- | --- | --- | --- | --- |
| 1 | A legacy server with the same package version is shut down before the current CLI opens a plan | `same-version legacy server is replaced before a canvas opens` | End to end | PASS |
| 2 | Health exposes package, protocol, and exact Canvas runtime identity | `GET /health identifies the app and version` | Integration | PASS |
| 3 | Mermaid remote enhancement starts only after document load | `a plan containing mermaid serves the themed Mermaid loader` | Integration | PASS |
| 4 | Open, browser load, await, feedback, reply, approval, reopen, end, and stop still work together | `tests/integration/plan-canvas-e2e.test.js` | End to end | PASS |
| 5 | The large ECC 2 to ECC 3 master plan bootstraps under blocked local storage | Headless Chrome DOM and screenshot check against `/canvas/24af75d4c4fe` | Browser | PASS |

## Coverage and full-suite evidence

`npm run coverage` passed all 3,993 discovered tests with these project totals:

- Statements: 88.98%
- Branches: 80.66%
- Functions: 94.32%
- Lines: 88.98%

The `scripts/lib/plan-canvas` group reached 98.38% statements, 88.2% branches,
98.64% functions, and 98.38% lines. Focused ESLint checks passed for every
modified JavaScript test and production file.

## Browser evidence and known gaps

The live shared server was initially process `15351`, started from the older
`ecc-tiered-sandbox` worktree, and served an unguarded `localStorage` client even
when invoked from current main. The patched CLI replaced it with the isolated
worktree server and restored persisted sessions. Two real plan sessions then
opened successfully. Headless Chrome with local storage disabled completed the
large master-plan DOM load in about two seconds and produced a rendered Canvas
screenshot.

Chrome was exercised directly on macOS. Safari and Firefox were not run. The
fix relies only on standard health JSON, dynamic `import()`, and the standard
window `load` event.
