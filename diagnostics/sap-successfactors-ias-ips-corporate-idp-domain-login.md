# Diagnostic Runbook: SAP SuccessFactors IAS/IPS — Corporate IdP Login Failure by Email Domain

**SuccessLabs Academy | Infrastructure Architecture (InfraA) | Support / Secure / Assure Labs**  
**Case type:** Authentication / SSO / Identity federation  
**Status:** Diagnostic guidance; root cause not confirmed  
**Priority:** UAT-blocking if affected users cannot log in  
**Last documented:** 2026-11-23

> This is a sanitized learning case. Email addresses and domain names below are illustrative. Do not publish real tenant URLs, SAML assertions, tokens, user identifiers, or sign-in logs without redaction.

## 1. Problem statement

SAP SuccessFactors is integrated with SAP Cloud Identity Services — Identity Authentication (IAS) and Identity Provisioning (IPS). A user from one email domain can sign in, while two users from other email domains receive: **“Sorry, we could not authenticate you. Try again.”**

**Confirmed facts:** Both failing users exist in IAS; authentication uses a corporate Identity Provider (IdP). A working account can authenticate. **Not yet established:** exact failure layer, domain restrictions, SAML mappings, corporate IdP product, or provisioning health.

| Test identity (illustrative) | Domain | Observed result |
| --- | --- | --- |
| user1@domain.com | domain.com | Successful |
| user2@domain-test.com | domain-test.com | Failed |
| user3@domain-test-fg.com | domain-test-fg.com | Failed |

## 2. Authentication flow and responsibilities

1. Browser accesses SAP SuccessFactors (service provider).
2. SuccessFactors redirects authentication to IAS.
3. IAS application conditional authentication selects the corporate IdP.
4. Corporate IdP authenticates the user and returns the assertion to IAS.
5. IAS applies configured identity federation, user matching, and assertion transformations.
6. IAS sends the SAML response to SuccessFactors.
7. SuccessFactors resolves the SAML identifier to an active application user.

**IAS authenticates/federates; IPS provisions identities.** Do not assume an IPS defect from an authentication error. An IAS user record existing does not prove corporate IdP sign-in or SuccessFactors user matching.

## 3. Non-disruptive diagnostic sequence

### D01 — Reproduce and collect evidence
- Use a fresh private browser session and the same SuccessFactors entry URL for working and failing users.
- Capture UTC timestamp, environment, correlation/request ID if displayed, error stage, and whether corporate IdP login is reached.
- Do not record passwords, tokens, or unredacted assertions in GitHub.

### D02 — IAS application routing
- IAS Administration Console → **Applications & Resources → Applications → [SuccessFactors application] → Trust → Conditional Authentication** (labels may vary by tenant).
- Compare default IdP and rule precedence for the working and failing email domains.
- Check whether rules depend on email domain, user group, or another attribute.
- If a domain rule is absent, verify that the corporate IdP actually accepts that domain before adding a narrowly scoped rule. Preserve the working route.

### D03 — IAS corporate IdP federation
- IAS → **Identity Providers → Corporate Identity Providers → [configured IdP] → Identity Federation**.
- Inspect user-store matching, permitted-user settings, and allowed email domains *if those controls are configured*.
- Verify IAS users are active, unlocked, and have correct login names and identity attributes. IAS email and login name need not match.
- Avoid broad allow-all changes.

### D04 — Corporate IdP checks
- Ask IdP administrator to locate sign-in attempts by user and timestamp.
- Verify account enabled, app assignment, guest/external account status, domain recognition, conditional access/MFA and claims.
- For Microsoft Entra ID: Enterprise applications → relevant federation application → Users and groups, Single sign-on → Attributes & Claims, and Sign-in logs.
- If rejected here, fix the corporate IdP issue before changing SuccessFactors or IPS.

### D05 — IAS logs
- IAS Administration Console → **Monitoring & Reporting → Troubleshooting Logs** (navigation may vary).
- Correlate one failing attempt to the timestamp. Determine whether IAS receives the request, selects the expected IdP, and receives a successful assertion.
- No IAS event: investigate redirect/tenant/application URL. IAS event with IdP failure: investigate routing or IdP. IAS success followed by SF rejection: inspect outgoing SAML and SF mapping.

### D06 — SuccessFactors SAML user mapping
- Inspect IAS application **Trust → Single Sign-On** subject identifier and assertion mapping.
- Compare the corporate IdP NameID, IAS login name, IAS outgoing NameID / UserName, and actual SuccessFactors `username` for working and failing users.
- The assertion's effective user identifier must resolve to the intended active SuccessFactors user. Do **not** assume the email address equals the SF username.
- Check relevant SuccessFactors login / SAML diagnostic records and account status.

### D07 — IPS provisioning (only if evidence indicates a mismatch)
- IPS → Source Systems / Target Systems → Provisioning Jobs and logs.
- Check skipped users, filters, attribute transformations, duplicate identities and target mappings.
- Rerun only the validated affected scope after fixing a confirmed provisioning issue.

## 4. Decision matrix

| Evidence | Likely failure boundary | Action |
| --- | --- | --- |
| No IAS event for login attempt | Entry URL / redirect / wrong IAS tenant | Validate application URL and trust configuration |
| IAS selects unexpected IdP | Conditional authentication | Fix targeted rule or precedence |
| Corporate IdP sign-in denied | Corporate IdP | Fix assignment, account, policy, or domain acceptance |
| IAS rejects federated identity | IAS identity federation | Fix IAS user matching or applicable restrictions |
| IAS authentication succeeds; SF rejects | SAML subject / SF account | Correct mapping to active SF username |
| User/attribute absent due to IPS filters | Provisioning | Correct filter/transformation; controlled reprovisioning |

## 5. UAT verification

- [ ] Working-domain user still authenticates.
- [ ] Both previously failing users authenticate through intended corporate IdP.
- [ ] IAS troubleshooting logs show expected authentication result.
- [ ] Effective SAML identity matches active SuccessFactors user.
- [ ] All three reach SuccessFactors homepage and expected authorized landing page.
- [ ] No regression from conditional-authentication changes.
- [ ] Redacted test evidence and timestamps recorded.
- [ ] Root cause and exact fix documented before closing the linked issue.

## 6. Change controls

Back up current IAS application settings and record change owner, test window, rollback procedure, and approvals. Change only the identified failing layer; do not disable MFA, weaken Conditional Access, or globally relax domain restrictions as a workaround.

## 7. Official reference entry points

- [SAP Cloud Identity Services documentation](https://help.sap.com/docs/cloud-identity-services)
- [SAP SuccessFactors platform documentation](https://help.sap.com/docs/successfactors-platform)

Use current tenant-specific SAP Help guidance and applicable SAP KBAs to confirm exact navigation and SAML settings before production changes.

## 8. Resolution record (complete after testing)

| Field | Value |
| --- | --- |
| Confirmed root cause | Pending |
| Failing authentication boundary | Pending |
| Configuration before/after | Pending |
| Change approval | Pending |
| UAT retest evidence | Pending |
| Resolution date | Pending |
| Related GitHub Issue | To be linked |

**Reusable principle:** In federated SuccessFactors authentication, isolate the failure across browser → IAS routing → corporate IdP → IAS federation → SAML subject → SuccessFactors account before modifying provisioning or trust configuration.
