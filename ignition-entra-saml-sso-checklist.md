# Ignition Gateway + Entra ID SAML SSO — Implementation Checklist

**Assumes:** Entra Enterprise Application already exists (SAML-based SSO type), nothing else configured yet.
**Tested with:** Ignition 8.3 and Microsoft Entra ID (free tier, security-group claims)

---

## 1. Gather Ignition's SP values

Before touching Entra, get the target gateway's own SP identity:

- **Identifier (Entity ID):** `https://<gateway-fqdn>:<port>`
- **Reply URL (ACS URL):** `https://<gateway-fqdn>:<port>/data/federate/callback/saml`

Confirm exact values from the gateway's own SP metadata:
```
https://<gateway-fqdn>:<port>/data/saml/metadata/sp
```
Always use the FQDN (not localhost) so other devices on the network can complete login.

---

## 2. Configure Entra Enterprise App — Basic SAML Configuration

- **Identifier (Entity ID):** paste value from Step 1
- **Reply URL (ACS):** paste value from Step 1
- Save

---

## 3. Configure Entra — SAML Certificates (Signing Option)

Go to **SAML Certificates → Edit**.

- Set **Signing Option** to **"Sign SAML response and assertion"** (not response-only, not assertion-only)
  - Ignition's parser expects a `<ds:Signature>` directly under `<samlp:Response>`, in addition to the one under `<Assertion>`. Response-only signing scope will fail with:
    ```
    SAMLDocumentParserException: Expected one Signature candidate for signed node 0
    and xPath /samlp:Response but got 0
    ```
- Leave **"Require verification certificates"** unchecked — Ignition's SP does not sign outbound AuthnRequests (`AuthnRequestsSigned="false"` in its metadata), so enabling this breaks SP-initiated login and IdP-initiated "Test" logins alike.
- Copy the **App Federation Metadata URL** — this is what you'll import into Ignition (includes IdP endpoints + signing cert in one shot).

---

## 4. Configure Entra — Attributes & Claims

Default claims (givenname, surname, emailaddress, name, displayname, NameID) are sufficient for ID/Username/Email/First/Last Name mapping — no changes needed unless customizing.

**If Roles/Security Levels are needed (Groups claim):**
- **Add a group claim**
- Under "which groups": choose **Security groups** (free tier — do NOT choose "Groups assigned to the application," which requires P1/P2)
- Source attribute: **Group ID** (GUID) — display name emission requires app-assignment mode (P1/P2 licensed)
- Note: Entra caps claim emission at 150 groups per token — not a practical concern at this scale

**Users/Groups assignment:**
- Confirm the target user(s) are assigned to the Enterprise App — Entra blocks sign-in for unassigned users even with a fully correct SAML config
- If using group-based Roles, confirm user is a *member* of the relevant security group(s) (separate step from app assignment)

---

## 5. Ignition Gateway — Create Identity Provider

**Config → Security → Identity Providers → Create New → SAML**

- Name it clearly (e.g., `EntraId`)
- Import via the **App Federation Metadata URL** from Step 3 (auto-populates IdP SSO endpoint + signing cert — cleaner than manual entry)
- Confirm the ACS URL displayed here matches Step 1

---

## 6. Ignition Gateway — User Attribute Mapping

All paths below assume default Entra claim URIs. Root: `authnResponse`. Prefixes: `saml2p:` / `saml2:`. All string values need trailing `/text()`.

| Field | Type | Source | Path |
|---|---|---|---|
| ID | direct | Authentication Response | `/saml2p:Response/saml2:Assertion/saml2:Subject/saml2:NameID/text()` |
| Username | direct | Authentication Response | `/saml2p:Response/saml2:Assertion/saml2:Subject/saml2:NameID/text()` |
| Email | direct | Authentication Response | `/saml2p:Response/saml2:Assertion/saml2:AttributeStatement/saml2:Attribute[@Name='http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress']/saml2:AttributeValue/text()` |
| First Name | direct | Authentication Response | `/saml2p:Response/saml2:Assertion/saml2:AttributeStatement/saml2:Attribute[@Name='http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname']/saml2:AttributeValue/text()` |
| Last Name | direct | Authentication Response | `/saml2p:Response/saml2:Assertion/saml2:AttributeStatement/saml2:Attribute[@Name='http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname']/saml2:AttributeValue/text()` |
| Roles (if using groups) | direct | Authentication Response | `/saml2p:Response/saml2:Assertion/saml2:AttributeStatement/saml2:Attribute[@Name='http://schemas.microsoft.com/ws/2008/06/identity/claims/groups']/saml2:AttributeValue/text()` |

---

## 7. Test Login

Use the IdP config's **Test Login** button. Review all three result tabs:

- **IdP Response Data** — confirms the raw parsed structure (namespace prefixes as Ignition sees them)
- **Mapped User Attributes** — confirms id/userName/firstName/lastName/email/roles resolved to real values, not blank
- **Security Level Grants** — confirms which Security Levels the session actually resolves to (`Authenticated` alone at minimum; group GUIDs will appear here as auto-generated Role nodes if Roles was mapped)

---

## 8. Authorization — pick ONE path per gateway

**Path A — No Groups claim / small user set:**
Use **User Grants** (Config → Security → Identity Providers → [provider] → User Grants) to manually assign specific users (by ID/email) to Security Levels. Does not require any Roles mapping. Does not scale well past a handful of users — no bulk/group assignment.

**Path B — Groups claim configured (Step 4):**
- Auto-generated GUID nodes appear under `Authenticated → Roles` after a live login with a matched claim
- Keep a mapping note somewhere (GUID → group display name) since Ignition only ever shows the raw GUID
- Do **not** try to manually check the built-in reserved `Administrator` node under Roles — it's system-generated and only satisfied by a literal `"Administrator"` string in a claim, which GUID-based emission will never produce
- Build custom Security Levels under `Authenticated → Roles` if you want a human-readable name distinct from the raw GUID node

---

## 9. Apply the Identity Provider to actual auth surfaces

Mapping/grants alone don't change how anyone logs in — you still need to wire the IdP into:

- **Gateway → Config → Security → General** — set **System Identity Provider** to this provider; decide on **"Always ask the IdP to re-authenticate users by default"** (checked = no SSO reuse, forces fresh Entra prompt each time — useful while testing, revisit for production)
- **Designer Authentication Strategy** — set to **Identity Provider**; select which Security Levels are required (ALL vs. AT LEAST ONE) for Designer access
- **Gateway Read/Write permissions** — same Security Level selection, separate screen from Designer

---

## 10. Known gotchas checklist (recap)

- [ ] Signing option set to "Sign SAML response and assertion" (not default)
- [ ] Verification certificates left OFF
- [ ] User/group actually assigned in Entra (not just app-registered)
- [ ] Entity ID / ACS URL use FQDN, not localhost, and match exactly on both sides
- [ ] XPath expressions end in `/text()`
- [ ] Roles claim only useful if consuming app (Ignition) is actually configured to read it — confirm via Test Login before assuming it's live
