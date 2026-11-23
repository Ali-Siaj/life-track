# Security and privacy

[← Back to the overview](../README.md)

Life Track holds personal data (money, health, body measurements), so security was part of the design from the first milestone, not something added later.

## Authentication

- **Supabase Auth** with cookie-based sessions. Public sign-ups are switched off; accounts are created by the owner.
- **Two checks on every page.** Middleware checks the session and refreshes the token before the page runs, and the app's server layout checks the user again.
- **Login cookies can't be read by page scripts.** "Stay signed in" keeps a login for 400 days; without it, the login ends when the browser closes. "Log out" ends only this device's session, and Settings has "Log out on all devices".
- **Password changes need the current password.** The app never stores a password; the login form is marked up for the browser's password manager.

## Data isolation: row-level security

The browser only ever holds Supabase's *publishable* key, which is safe to expose because the database itself enforces who can see what. All 31 tables have row-level security turned on, with policies that allow a row to be read, inserted, updated or deleted only when its `user_id` equals the signed-in user's id. The secret service key is never used by the app.

## API keys in Supabase Vault

Each user saves their own Hevy and Gemini keys in Settings. They are kept in **Supabase Vault** (encrypted at rest) and handled only through three database functions: save, read and remove. Each function acts only on the key of the signed-in user, signed-in users have no access to the Vault schema, and they cannot write the key table directly. A key is never logged, never shown again after saving, never sent to the browser, and left out of the data export on purpose.

I verified this in a **rolled-back dry run on the real database with two users**: one user could not read, replace or remove the other's key, direct writes to the key table were refused, signed-in users could not reach the Vault schema, and visitors who were not signed in could not call the functions at all.

An earlier design encrypted the keys with a server secret. I replaced it with Vault because no user could save a key until that secret had been set by hand on the hosting side.

## Who may use which key

A request uses the user's own saved key. Only accounts on an allow-list (which no user can add themselves to: no write grant, no write policy) may fall back to a key saved on the server. If a saved key can't be read, the server's key is **not** used as a stand-in, so one user's workouts can never be pulled into another user's account.

## Browser hardening

Every response carries security headers: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, a `Content-Security-Policy` that forbids framing, plugins and foreign form targets, a strict `Referrer-Policy`, and a `Permissions-Policy` that allows only the camera and microphone the app itself uses.

## What leaves the app

- **Gemini** receives only the sentence typed (or the budget lines of a pasted day), the category names and the date. The manual forms never use it.
- **"Copy for Gemini"** builds a plain-text summary of recent weeks for my own Gemini chat; it leaves money data out.
- **Secrets never reach the repository.** Local keys live in an ignored environment file, and production keys are runtime secrets on the hosting platform.
