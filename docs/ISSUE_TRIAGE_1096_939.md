# Issue triage — #1096, #939 (not reproducible)

Findings from triaging the four reported issues in this repository. #1097 and
#1094 were real and are fixed in the previous commit. The two below do not
reproduce against the current code; the evidence is recorded here so it is not
re-derived from scratch.

## #1096 — `submit_attestation` arity in `pause_tests.rs`

**Reported:** `submit_attestation` takes 6 arguments after `env`; both call
sites in `src/pause_tests.rs` (lines 86 and 131) pass 5, so the file does not
compile and the pause/unpause gate on attestation has no coverage.

**Actual:** the signature at `src/contract.rs:798` is:

```rust
pub fn submit_attestation(
    env: Env,
    issuer: Address,
    subject: Address,
    timestamp: u64,
    payload_hash: Bytes,
    signature: Bytes,
) -> u64
```

Five parameters after `env`, no `metadata` argument, and a `u64` return rather
than a `Result`. The word `metadata` does not appear anywhere in
`src/pause_tests.rs`. The two call sites named in the report pass exactly five
arguments, so they match the contract.

`cargo check --all-targets` confirms it. The `E0061` arity errors appear in
different modules:

```
src/attestation_pagination_tests.rs:72, 100, 141, 142, 145, 176, 207, 227, 259, 323, 326, 354
  error[E0061]: this method takes 5 arguments but 6 arguments were supplied
```

Those call sites pass a sixth `&None` argument. `pause_tests.rs` raises no such
error, so the file the report calls broken compiles.

There is a real arity defect here, but it is the mirror image of the one
reported: it is in `attestation_pagination_tests.rs` (and likely
`new_features_tests.rs` and `load_simulation_tests.rs`, which pass an explicitly
typed `&None`), not in `pause_tests.rs`. It is not fixed by this change.

## #939 — `ApiRequestPanel` History feature coverage

**Reported:** no test exercises `persistHistory`, the expand/collapse toggle,
the "Clear history" button, or the `localStorage` read/write behaviour in
`loadHistory`/`saveHistory`.

**Actual:** `ui/components\ApiRequestPanel.test.tsx` has a "History Persistence"
describe block and a "History Display and Interaction" describe block
containing eight tests. Every scenario the report lists as missing is asserted:

| Scenario | Test |
|---|---|
| entry written to `localStorage`, endpoint read back | line 162 |
| `persistHistory={false}` leaves `localStorage` null | line 206 |
| 20-entry cap holds across 25 rerenders | line 179 |
| toggle hides and shows the history list | line 222 |
| entry renders with method and endpoint | line 251 |
| "Clear history" empties `localStorage` | line 271 |
| failed request recorded and flagged | line 289 |

The report's claim that there is "not a single reference to history,
`localStorage`, or `persistHistory` anywhere in the test file" is not accurate.

## Build state

`cargo check --all-targets` reports **142 errors in the test build**. The arity
class is described above. The remainder is pre-existing and unrelated to these
issues: a `std` reference inside this `no_std` crate
(`src/attestor_cap_batch_tests.rs`) and stale Soroban test-helper usage in
`src/attestor_event_tests.rs` (`Address::random`, `Events::publish_expect`, and
an `AttestorRegistered` symbol that no longer exists). None of it is addressed
by this change.

`cargo test` was not run: linking test binaries needs a host linker, which is
unavailable in this environment. The tests remain unexecuted, and CI's test job
will stay red until those 142 errors and the missing linker are dealt with.
