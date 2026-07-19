---
title: "Implementing SSO in a Node.js App: What, Why, and How"
description: "A hands-on walkthrough of adding Google-based Single Sign-On to an Express app, including the mistakes I made along the way."
pubDate: 2026-07-19
type: TUTORIAL
tags: [ "sso", "oauth2", "oidc", "nodejs", "express", "passport" ]
author: "devanshb"
---

# Implementing SSO in a Node.js App: What, Why, and How

Single Sign-On (SSO) is one of those things that feels like magic from the outside and surprisingly simple once you've wired it up yourself. This post walks through exactly how I added "Login with Google" to a small Express app — what SSO actually is, why I built it this way, and the step-by-step process I followed, mistakes included.

## What is SSO?

SSO lets a user log into your app using an identity they already have somewhere else — Google, GitHub, Okta, Azure AD — instead of creating a new username and password just for you.

What I built specifically is **OAuth 2.0 + OpenID Connect (OIDC)** via Google. In plain terms: my app never sees the user's Google password. Instead, Google vouches for who the person is, and hands my app a token confirming that identity.

## Why build it this way

A few reasons I went with Google OAuth instead of rolling my own auth system:

- **No password storage.** I never have to hash, store, or worry about leaking passwords.
- **Faster login for users.** One click instead of a signup form.
- **Trust offloaded to Google.** Google already handles MFA, breach detection, and account recovery — I don't have to.
- **It's the standard pattern.** Almost every SaaS product you've used does some version of this.

On top of that, I wanted to restrict access to a specific organization — only people with a `@one2n.in` email should be able to log in. That's a common real-world requirement: internal tools, admin panels, and company dashboards usually aren't meant to be public.

## How it works, conceptually

Before the code, it's worth understanding the actual handshake — this is the part that makes everything else make sense.

1. User clicks **Login with Google** on my app.
2. My app redirects them to Google with a `client_id`, `redirect_uri`, and requested `scope` (`profile email`).
3. User logs in (or is already logged in) and approves access.
4. Google redirects back to my app's callback URL with a one-time `code`.
5. My **backend** exchanges that code — plus my `client_secret` — directly with Google's server for an identity token.
6. My app reads the user's email/name from that token, checks whether they're allowed in, and creates a session for them.

Here's that full flow, end to end:

![SSO traffic flow diagram showing browser, Google, and app interactions](./sso-flow-diagram.svg)

The key thing to notice: Google's part of the flow ends the moment my app gets the identity back. Everything after that — staying logged in, protecting pages — is my own app's session, not Google's.

## Step-by-step: how I built it

### Step 1 — Set up Google OAuth credentials

In [Google Cloud Console](https://console.cloud.google.com/), I created a project, configured the OAuth consent screen (External, added myself as a test user), then created an **OAuth client ID** under Credentials:

- Application type: Web application
- Authorized redirect URI: `http://localhost:3000/auth/google/callback`

This gave me a Client ID and Client Secret.

### Step 2 — Structure the project

Rather than cramming everything into one file, I split it by responsibility from the start:

```
sso-demo/
├── server.js                # entry point
├── config/
│   └── passport.js          # Google strategy + session serialization
├── routes/
│   ├── auth.js               # /auth/google, /auth/google/callback, /auth/logout
│   └── index.js               # / and /profile
├── middleware/
│   └── ensureAuthenticated.js  # route guard
├── models/
│   └── User.js                 # find-or-create user store
└── views/
    ├── home.ejs
    └── profile.ejs
```

This structure paid off almost immediately — when I later added the domain restriction, it was a change in exactly one file (`config/passport.js`), not a rewrite of the whole app.

### Step 3 — Install dependencies

```bash
npm install express express-session passport passport-google-oauth20 dotenv ejs
```

### Step 4 — Configure the Google strategy

In `config/passport.js`, I set up the strategy with my credentials and a callback that runs once Google has already confirmed the user's identity:

```js
passport.use(
  new GoogleStrategy(
    {
      clientID: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      callbackURL: `${process.env.BASE_URL}/auth/google/callback`
    },
    (accessToken, refreshToken, profile, done) => {
      const user = User.findOrCreateFromGoogleProfile(profile);
      return done(null, user);
    }
  )
);
```

### Step 5 — Build the auth routes

```js
router.get('/google', passport.authenticate('google', { scope: ['profile', 'email'] }));

router.get(
  '/google/callback',
  passport.authenticate('google', { failureRedirect: '/' }),
  (req, res) => res.redirect('/profile')
);
```

`passport.authenticate` handles the code exchange automatically — I never manually touch Google's token endpoint.

### Step 6 — Wire up sessions

`express-session` plus `passport.serializeUser`/`deserializeUser` keep the user logged in across requests using a cookie, without hitting Google again on every page load.

### Step 7 — Restrict login to my organization's domain

This was the part specific to my use case. Google confirms *who* someone is — it has no concept of whether they're *allowed* into my app. That check has to happen in my own code, right after the identity comes back:

```js
const email = profile.emails?.[0]?.value || '';
const allowedDomain = process.env.ALLOWED_EMAIL_DOMAIN; // "one2n.in"

if (allowedDomain && !email.toLowerCase().endsWith(`@${allowedDomain.toLowerCase()}`)) {
  return done(null, false, { message: `Access restricted to @${allowedDomain} accounts.` });
}
```

I also added the `hd` parameter on the initial redirect, which nudges Google's account picker toward the right domain — but that's a UX hint only, not real enforcement. The `endsWith` check above is the actual gate.

### Step 8 — Test locally, then share via ngrok

Locally, everything worked against `localhost:3000`. To let others try it, I used ngrok:

```bash
ngrok http 3000
```

This gave me a public HTTPS URL. Two things had to change to make SSO work through it:

- Add the ngrok URL as a new **Authorized redirect URI** in Google Cloud Console (`https://your-url.ngrok-free.dev/auth/google/callback`), alongside the localhost one.
- Set `BASE_URL` in `.env` to the ngrok URL, so my app builds the correct absolute callback URL instead of defaulting to `localhost`.

## What actually went wrong (and what it taught me)

Three real errors came up while building this — worth documenting because they're all common:

**`TypeError: OAuth2Strategy requires a clientID option`**
Cause: `.env` wasn't loaded, or the Client ID was still the placeholder text from `.env.example`. Fix: confirm `.env` exists and has real values, not the example strings.

**`Error 400: redirect_uri_mismatch`**
Cause: the redirect URI my app was sending didn't match what was registered in Google Cloud Console — happened when I switched from `localhost` to an ngrok URL without updating either side. Fix: keep the registered URI and the app's `callbackURL` in exact sync, including protocol (`http` vs `https`).

**Domain check blocking my own login**
Cause: I'd set `ALLOWED_EMAIL_DOMAIN=one2n.com` when my actual email was `@one2n.in` — a typo in the config, not a bug in the logic. Fix: added temporary debug logging to print exactly what email Google sent back versus what my app expected, which made the mismatch obvious immediately.

The pattern across all three: nothing was actually broken in the OAuth flow itself — every failure was a mismatch between what was configured and what was expected. Worth remembering next time something in an SSO setup "doesn't work": check the exact strings first.

## Key takeaways

- SSO via OAuth2/OIDC means **outsourcing identity verification**, not authorization — your app still has to decide who's allowed in.
- Keep provider-specific config (`config/passport.js`) separate from your app's routes — it makes extending to a second provider, or adding access rules, a contained change.
- Sessions, once created, don't need the identity provider again — that's your own app's job from that point on.
- Most SSO bugs aren't protocol bugs. They're configuration mismatches: wrong redirect URI, unset env var, typo'd domain. Verify exact strings before assuming the flow itself is broken.

## What's next

From here, the natural extensions are: swapping the in-memory user store for a real database, adding a second provider like GitHub, and comparing this against SAML — the XML-based, more enterprise-flavored SSO protocol used by tools like Okta and Azure AD.
