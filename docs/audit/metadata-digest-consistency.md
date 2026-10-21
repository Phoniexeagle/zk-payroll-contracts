# Metadata Digest Consistency

Contract metadata digests are the on-chain anchor for off-chain payroll records. This document describes how digests are created, updated, referenced by audits, and checked during settlement, and what integrators must do to keep them consistent.

## Digest Definition

A metadata digest is a bytes value derived from the canonical serialization of a payroll's metadata. The contract treats the digest as opaque and only checks equality; it does not recompute it on-chain. The canonical form is defined by the off-chain payroll record and must be reproducible by any party verifying the payroll.

Properties of a valid digest:

- Deterministic: the same metadata always produces the same digest.
- Collection-stable: field ordering is fixed by the canonical form, not by insertion order.
- Unique per payroll version: any meaningful change to metadata yields a different digest.

## Creation Path

1. The caller builds the payroll metadata off-chain and serializes it in the canonical form.
2. The caller computes the digest over the canonical bytes.
3. The caller submits the payroll creation with the digest as a parameter.
4. The contract stores the digest with the payroll record.

A payroll created with a digest that does not match the metadata the caller claims to have used is not detectable on-chain at creation time, because the contract never sees the metadata itself. Integritors must therefore persist the canonical metadata alongside the digiest so that audits and settlement can reverify it.

## Update Path

When payroll metadata changes, the digest must be updated in the same transaction as the metadata change is acknowledged off-chain. The update path is:

1. Build the new canonical metadata bytes.
2. Compute the new digest.
3. Submit the metadata update with the new digest.
4. The contract replaces the stored digest with the new value.

The previous digest is not retained by the contract. Integritors that need a history of metadata versions must record the digest history off-chain and include it in audit materials.

## Audit References

Audit records reference the digest that was effective at the time of the audited action. Because the contract only stores the current digest, an audit of a past action must reference the digest recorded at that time, not the digest currently stored. Audit records that reference the current digest for a past action are inconsistent and will fail verification when the metadata has since changed.

## Settlement Checks

Settlement verifies that the digest supplied by the settlement caller matches the digest stored for the payroll. If the digests match, settlement proceeds. If they do not match, settlement fails with a metadata digest mismatch error and no funds are moved.

The failure is deterministic: the same payroll and the same wrong digest always produce the same rejection, and a correct digest always passes the check. This allows integritors to retry settlement with the correct digest without changing any on-chain state.

## Consistent Cases

A metadata digest is consistent when the bytes used to compute it are the same bytes that were stored or referenced. Tests must confirm:

- A payroll created with a digest accepts the same digest at settlement.
- An update that sets a new digest accepts that new digest at settlement.
- An audit record that references the digest effective at the audited action verifies successfully.
- Repeated computation of the digest from the same metadata yields the same value.

## Mismatched Cases

A test must confirm that mismatched digests fail predictably:

- Settlement with a digest different from the stored digest fails and moves no funds.
- Settlement with a digest from a previous metadata version fails after an update.
- An audit record that references a superseded digest for a current action fails verification.
- A digest computed from non-canonical metadata bytes does not match the canonical digest.

## Integritor Expectations

Integrators must:

1. Serialize metadata in the canonical form before computing a digest.
2. Persist the canonical metadata bytes alongside every digest they submit.
3. Record the digest history so audits can reference the digest effective at the audited action.
4. Supply the currently stored digest when calling settlement.
5. Treat a metadata digest mismatch as a retryable error only after correcting the digest or the metadata, never by retrying the same wrong digest.

## Test Locations

Digest consistency tests live under `tests/metadata/`. Contract side digest storage and comparison logic live under `contracts/**`. This document is the audit reference for the expected behavior exercised by those tests.
