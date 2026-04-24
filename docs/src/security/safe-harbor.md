# Safe harbour

Security researchers acting in good faith within the scope of this page
will not face legal action from Zhilicon Semiconductors.

## Scope of the safe harbour

Zhilicon commits, under the good-faith standard established by [CFAA
cases and the DMCA security-research exemption](https://www.federalregister.gov/documents/2021/10/28/2021-23251/exemption-to-prohibition-on-circumvention-of-copyright-protection-systems-for-access-control), to
not pursue civil, criminal, or administrative action against researchers
who:

1. Test **only** against publicly released artefacts or reference
   designs — not against customer production deployments.
2. Access **only** the data necessary to demonstrate the vulnerability;
   do not enumerate or exfiltrate production customer data.
3. **Do not modify** or destroy data they encounter incidentally.
4. **Do not pivot** from the initial finding into other Zhilicon
   systems, customer systems, or third-party systems.
5. **Do not degrade** the availability of Zhilicon infrastructure
   (no DoS, no volumetric testing).
6. **Stop at the earliest point** that demonstrates the vulnerability.
   Additional exploitation "because we were curious" voids safe
   harbour.
7. Report the finding through the [disclosure portal](disclosure-portal.md)
   within **30 days** of initial discovery.
8. Honour the [coordinated disclosure window](cve-policy.md#coordinated-disclosure).

## Activities inside the safe harbour

- Static analysis of released SDK source, firmware images, and
  reference designs.
- Dynamic analysis of released artefacts running in the researcher's
  own environment.
- Analysis of publicly disclosed cryptographic primitives implemented
  in the Zhilicon silicon.
- Responsible exploitation of a vulnerability on the researcher's own
  hardware, up to the point that demonstrates the finding.
- Disclosure of the finding to CERT / ICS-CERT / national CSIRTs where
  such disclosure is required by local regulation.

## Activities **outside** the safe harbour

- Testing against **customer production deployments** — requires the
  customer's written consent.
- **Social-engineering** attacks against Zhilicon employees, contractors,
  or customers.
- **Physical-security** testing against Zhilicon facilities.
- **Denial-of-service** demonstrations against Zhilicon production
  infrastructure.
- **Mass-testing** — automated scanning, fuzzing at network scale — of
  production endpoints without prior written coordination with
  `security@zhilicon.com`.
- **Acquiring, exploiting, or selling** customer data incidentally
  accessed during research.
- Testing that violates applicable export-control, sanctions, or
  embargo law.

## What "good faith" means in practice

Good faith is assessed on two axes: **intent** (was the researcher
trying to help Zhilicon fix the problem?) and **conduct** (did the
researcher stay within the scope of activities listed above?). Good
intent with poor conduct can lose the safe-harbour protection; good
conduct with malicious intent (e.g. demanding payment as a condition of
disclosure) also loses it.

Zhilicon gives the benefit of the doubt — if the researcher's intent is
ambiguous but their conduct followed the scope, we will extend the
safe harbour and pursue the finding as a report, not a threat.

## Coordination with other programmes

The Zhilicon disclosure programme honours safe-harbour commitments
issued by:

- **National CSIRTs** — US-CERT, CERT-EU, JPCERT, CNCERT, etc.
- **Industry CSIRTs** — ICS-CERT for the industrial slice of our
  customer base.
- **Bug-bounty platforms** — HackerOne, Bugcrowd, Intigriti — *for
  reports that originate through their coordination channel*. Reports
  submitted outside those channels fall under the disclosure-portal
  safe harbour on this page.

## Legal affirmation

This page is a public commitment. A researcher who acts in accordance
with it and who later faces adverse action can cite this page.
Zhilicon's Legal team reviews the safe-harbour text annually.

## Related

- [Disclosure portal](disclosure-portal.md)
- [CVE policy](cve-policy.md)
