# PGP key management

The `security@zhilicon.com` PGP key is referenced from
[`.well-known/security.txt`](../../../.well-known/security.txt) and from
the disclosure portal's encrypt-in-browser path. This page is the
authoritative procedure for generating, publishing, and rotating that
key.

## Publication

The public key lives at a stable HTTPS URL:

> `https://zhilicon.com/.well-known/security-pgp-key.asc`

The `security.txt` file references this URL via its `Encryption:` field.
The URL is versioned implicitly — a rotation replaces the content at the
same URL. Fingerprint continuity is broken deliberately at each
rotation; see [rotation](#rotation) below.

## Current fingerprint

> **2026-04-17 — fingerprint pending key generation.**
>
> The production PGP key is generated on a Zhilicon-managed hardware
> security module (HSM) during the company-wide security-bootstrap
> ceremony. Until that ceremony completes, the disclosure portal
> operates through the plain-text mailto channel only, with an explicit
> banner on the form informing reporters that transport-layer TLS is
> the sole confidentiality guarantee until the PGP key lands.

Once generated, this page is updated by a PR that touches only the
**Current fingerprint** section above, reviewed by the Security COE, and
co-signed by the CSO. The PR cannot be squash-merged — the full history
of fingerprints is a supply-chain artefact.

## Generation procedure

1. Generate a **primary certification key** offline, on an
   air-gapped laptop, under four-eyes supervision:

   ```sh
   gpg --full-generate-key
   #  Key type:            RSA and RSA   (4096 bits for both)
   #  Expiration:          3 years
   #  Real name:           Zhilicon Security Team
   #  Email:               security@zhilicon.com
   #  Comment:             Disclosure key vYYYY
   ```

2. Generate **three subkeys**:

   - Signing (`S`) — 4096-bit RSA or Ed25519
   - Encryption (`E`) — 4096-bit RSA or Curve25519
   - Authentication (`A`) — only if WebAuthN / SSH use is planned

3. **Move the primary key to the HSM** via `gpg --import` from the HSM
   operator session. The primary key never resides on a network-
   connected host.

4. Export only the subkeys plus the primary public material:

   ```sh
   gpg --export --armor security@zhilicon.com > security-pgp-key.asc
   ```

5. Publish `security-pgp-key.asc` at
   `https://zhilicon.com/.well-known/security-pgp-key.asc` with
   `Content-Type: application/pgp-keys`.

6. Update this page's **Current fingerprint** section.

7. Publish the fingerprint on at least two out-of-band channels
   (LinkedIn company page, conference presentation, customer portal)
   to provide cross-verifiable evidence of the rotation.

## Rotation

The key rotates:

- On scheduled cadence: every **24 months**, 6 months before expiry.
- On incident: within **72 hours** of a confirmed compromise indicator.

Rotation procedure:

1. Generate the next key on the air-gapped laptop using the generation
   procedure above.
2. Sign the new primary key with the outgoing primary key (key
   transition signature), where possible.
3. Publish the new `security-pgp-key.asc` at the stable URL.
4. Update **Current fingerprint** on this page and keep the previous
   fingerprint(s) in the **Rotation history** table below.
5. Update the `Expires:` field in
   [`.well-known/security.txt`](../../../.well-known/security.txt).
6. Notify registered customers via the per-zone operations channel.
   (The disclosure portal and the email path accept ciphertext signed
   under either the old or the new key for a 30-day overlap window.)

## Rotation history

| Fingerprint                                        | Valid from   | Valid to     | Notes            |
| -------------------------------------------------- | ------------ | ------------ | ---------------- |
| *(none yet — pending bootstrap ceremony)*          | —            | —            | Plaintext email + TLS-only interim |

## Customer verification

Customers who want to verify the fingerprint out-of-band:

1. Fetch `https://zhilicon.com/.well-known/security-pgp-key.asc`.
2. Run `gpg --with-fingerprint security-pgp-key.asc`.
3. Compare with the fingerprint published on this page.
4. If they disagree, assume compromise and contact Zhilicon through an
   independent channel (LinkedIn corporate page, a known partner
   contact, a customer success manager).

## Related

- [Disclosure portal](disclosure-portal.md) — consumes the published key.
- [CVE policy](cve-policy.md) — references the CNA relationship that
  depends on the key's chain of custody.
