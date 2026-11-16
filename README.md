# ignition-entra-saml-sso

Step-by-step checklist for configuring SAML single sign-on between **Microsoft Entra ID** and an **Inductive Automation Ignition 8.3** gateway.

## What's here

| File | Purpose |
|---|---|
| [`ignition-entra-saml-sso-checklist.md`](./ignition-entra-saml-sso-checklist.md) | The full implementation checklist, from Entra app config through Designer and Gateway authentication |

## Scope

- Assumes the Entra Enterprise Application already exists (SAML-based SSO type) and nothing else is configured.
- Covers the Ignition side as a SAML Service Provider: Identity Provider setup, user attribute mapping, Test Login, and Security Level grants.
- Covers both authorization paths: manual **User Grants** (no groups claim) and **Roles** driven by an Entra security-group claim.
- Works on the Entra free tier. Group claims use **Security groups**, not "Groups assigned to the application" (P1/P2).

## Gotchas the checklist calls out

- Entra's signing option must be **Sign SAML response and assertion**. Ignition expects a signature on the `Response` element itself.
- Leave **Require verification certificates** off. Ignition's SP does not sign its AuthnRequests.
- Ignition 8.3 XPath mappings use `saml2p:` / `saml2:` prefixes and need a trailing `/text()`.
- The built-in `Authenticated → Roles → Administrator` level only matches a literal `Administrator` role string. Entra emits group GUIDs by default, so it won't match unless you emit display names (which needs P1/P2).

## Placeholders

Replace `<gateway-fqdn>` and `<port>` with your gateway's values. Use a real FQDN, not `localhost`, so other devices can complete the login flow.
