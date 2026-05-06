# Ansible Certified Collection Signing & Publishing Pipeline

How a partner's Ansible collection gets from a GitHub tag to a signed entry on `console.redhat.com/ansible/automation-hub`. Useful when a customer asks "why is version X on Galaxy but not on Automation Hub yet?"

## TL;DR

Signing is **not a separate queue**. It happens automatically once a collection clears Red Hat's certification pipeline. If a version is missing from Automation Hub, it has not been submitted or it is still in certification — it is almost never "waiting to be signed."

Galaxy and Automation Hub are **independent destinations**. A release on Galaxy does not automatically flow to Automation Hub. The partner must submit to each.

## The two destinations

| Destination | Who controls publish | Typical lag from GitHub tag | Signed? |
|---|---|---|---|
| **galaxy.ansible.com** | Partner (open publish via their CI) | Minutes to hours | No |
| **console.redhat.com Automation Hub** | Partner submits → Red Hat certifies → Pulp signs on publish | Days to weeks | Yes (GPG, Red Hat key) |

## Automation Hub pipeline (the certified path)

1. **Partner uploads** a collection tarball to their partner namespace on `console.redhat.com`.
2. **Automated import checks** run on upload:
   - `ansible-test sanity`
   - lint
   - manifest / metadata validation
   - Failures reject the upload — partner must fix and resubmit.
3. **Red Hat Partner Engineering review** — joint-support readiness, policy compliance.
4. **Publish + automatic signing** — Pulp signs `MANIFEST.json` with Red Hat's GPG key using the configured signing script (`PULP_SIGNING_KEY_FINGERPRINT`). The collection appears as "Signed" in the Hub UI.

Partners can run the [partner-certification-checker](https://forum.ansible.com/t/introducing-the-red-hat-partner-certification-checker/45607) GitHub Action locally before submission to catch the import-check failures up front.

## Diagnosing "version X is on Galaxy but not on Automation Hub"

Almost always partner-side. In rough order of likelihood:

1. Partner has not submitted the version to their Automation Hub namespace yet.
2. Submission failed automated checks; partner needs to fix and resubmit.
3. Submission is in Partner Engineering review.
4. (Rare) Pulp signing infrastructure issue on Red Hat side.

**To confirm:** ask the partner directly, or open a Red Hat case so Partner Engineering can check the import queue for that namespace.

## Verifying signatures on the consumer side

Customers consuming signed collections via `ansible-galaxy` opt in to verification by importing Red Hat's public key into a keyring:

```bash
gpg --import --no-default-keyring --keyring ~/.ansible/pubring.kbx my-public-key.asc
ansible-galaxy collection install <ns>.<col> --keyring ~/.ansible/pubring.kbx
```

The server returns ASCII-armored detached GPG signatures over `MANIFEST.json`, which is then used to verify the collection contents.

## Public references

- [Ansible Certified Content FAQ](https://access.redhat.com/articles/4916901) — Galaxy vs. Automation Hub distinction
- [Red Hat Ansible Automation Certification (Partner Connect)](https://connect.redhat.com/en/partner-with-us/red-hat-ansible-automation-certification) — partner-facing entry point
- [Certification Workflow Guide (PDF)](https://connect.redhat.com/sites/default/files/2021-06/MBU%20_%20Ansible%20Certification%20Workflow%20Guide.pdf) — partner submission steps
- [Partner certification checker](https://forum.ansible.com/t/introducing-the-red-hat-partner-certification-checker/45607) — pre-submission GitHub Action
- [Collections and content signing in private automation hub (AAP 2.3)](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.3/html/managing_red_hat_certified_and_ansible_galaxy_collections_in_automation_hub/assembly-collections-and-content-signing-in-pah) — signing mechanics, signing script, keyring setup
- [Managing automation content (AAP 2.5)](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/managing_automation_content/managing-cert-valid-content) — current canonical doc
- [`ansible-sign` CLI rundown](https://docs.ansible.com/projects/sign/en/latest/rundown.html) — note: this is for *project* signing, not collection signing (commonly confused)
