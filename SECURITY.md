# Security Policy

## Practices

- **No credentials live in this repository.** Every API key, access token, and database password is stored only in the workflow engine's own encrypted credential store. Anything resembling a secret in this repo is a placeholder (`.example`) value, never a real one.
- **Least-privilege access tokens.** Each external service (the messaging API, the record store, the reasoning layer) is connected with a token scoped to only what that specific integration needs.
- **The database holding conversation state is not exposed publicly.** It's reachable only from the workflow engine itself, over an internal network.
- **Personal data in transit and at rest is limited to what registration actually requires.** See `PRIVACY.md` for the full list of what's collected and why.
- **The messaging channel's own delivery-status callbacks are explicitly filtered out** before any registration logic runs, so the system's own outbound message receipts can't be misinterpreted as user input.

## Data Handling

This system collects: a registrant's title, first and last name, the office they hold, their birth month and day, their phone number, their preferred language, and a photo. All of it is collected because it's the actual content of a member registration; none of it is collected incidentally or for any purpose beyond that. See `PRIVACY.md` for retention and third-party processing detail.

## Reporting a Vulnerability

If you find a security issue in this project, please open a private report rather than a public issue. Email **services@anthonychilaka.com** with details. We'll acknowledge within a reasonable timeframe and aim to address confirmed issues promptly.
