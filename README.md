

I found a SAML SSO bug that let a normal user take over another user's account in a different organization.

No phishing.
No victim interaction.
No stolen password.

Just one broken trust decision.

The app validated a SAML assertion using the attacker's organization SSO configuration, then looked up the asserted email globally and issued tokens for that user.
which mean i can sign a (claim) by what ever i want i could have done many things like leaking uuid of org or user -_- 

That meant my own tenant could become the identity provider for someone else's account.




## Where the Bug Started

The app had organization-level SAML SSO.

That is normal:

```text
Organization A has its own SAML config.
Organization B has its own SAML config.
Each organization should only authenticate its own users.
```

So I asked the question that usually matters in multi-tenant auth:

```text
What happens if one tenant's SSO config claims a user from another tenant?
```

That question became the bug.

## The Setup

I created two test accounts:

```text
Attacker account -> Organization A
Victim account   -> Organization B
```

Both accounts were mine. Nothing real was touched.

Then I configured SAML on the attacker organization using my own IdP certificate. That meant I controlled the key used to sign SAML assertions for my organization.

The suspicious part came next.

The app let my attacker organization add the victim's email domain to the SAML allowed-domain list.

At that point I had:

```text
My organization
My SAML certificate
My private signing key
Victim email domain allowed
```

That was enough to test whether the backend was binding SAML login to the correct tenant.

## The Moment It Clicked

First, I checked the boring controls.

Unsigned SAML responses failed.
Expired SAML responses failed.
Users that did not exist failed.
Wrong domains failed.

Good.

That meant this was not a cheap "SAML accepts anything" bug.

The signature checks were real. The SAML parser was doing work. The app was not completely broken.

But then I signed a valid SAML assertion for the victim email using my attacker-controlled IdP key and sent it through my attacker's organization SSO flow.

Expected result:

```text
No. This user does not belong to your organization.
```

Actual result:

```text
200 OK
access token
refresh token
token subject = victim user
```

That was the account takeover.

## What Went Wrong

The backend trusted two things separately, but never tied them together.

It checked:

```text
Was this SAML response signed by the IdP configured for this organization?
```

Yes.

Then it checked:

```text
Does this email belong to a real user?
```

Yes.

But it skipped the most important question:

```text
Does this user belong to the organization whose IdP signed the assertion?
```

No.

That missing check broke the tenant boundary.

The app was treating SAML like this:

```text
Valid signature + existing email = login
```

But in a multi-tenant app, it has to be:

```text
Valid signature + existing email + user belongs to this tenant = login
```

That one missing condition changed everything.

## Why This Was Critical

The returned token was not just a weird response.

I used it to request the victim's profile and organization context, and the backend treated me as the victim.

So the impact was real:

```text
Normal account -> attacker-controlled SAML config -> signed victim assertion -> victim session
```

A real attacker could access whatever the victim could access inside the app: documents, signature requests, contacts, templates, organization data, and connected workflows.

The refresh token made it worse because the access was not just a one-time response.

## The Clean Exploit Chain

The whole thing came down to this:

```text
1. Create a normal account.
2. Configure SAML on your own organization.
3. Add the victim's email domain as an allowed domain.
4. Sign a SAML assertion for the victim email.
5. Send it to the SAML login flow with your organization ID.
6. Receive tokens for the victim account.
```

That is the kind of bug I like hunting for: not loud, not flashy, just a broken assumption sitting in the auth logic.

## The Fix

The fix is simple in concept:

```text
After SAML validation, resolve the user only inside the organization that owns that SAML config.
```

If Organization A's IdP signs the assertion, the asserted user must be a member of Organization A.

Also:

- Do not let tenants claim arbitrary email domains without proving ownership.
- Treat organization IDs in login requests as untrusted.
- Prefer SP-initiated SAML with server-side state.
- Add a regression test where Tenant A signs a valid assertion for a Tenant B user. That test should never return tokens.

## Final Takeaway

SAML was not the real problem here.

The bug was trust confusion.

The app trusted the attacker's IdP to prove the assertion was valid, then trusted a global email lookup to decide who should get a token.

Authentication worked.
Authorization did not.

And in multi-tenant SSO, that difference is the whole game.
