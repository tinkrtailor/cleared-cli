# Cleared CLI

A native command-line client for [Cleared](https://clearedpay.xyz): manage customers,
issue and cancel invoices, check payments, and view your dashboard. No JavaScript
runtime, Node.js, Bun, Cargo, or MCP host is required to run the binary.

This repository distributes binaries and public usage documentation. The source code
is private. Download executables from [Releases](https://github.com/tinkrtailor/cleared-cli/releases);
GitHub's automatic “Source code” archives contain only this distribution repository.

## Install v0.2.0

Download the versioned installer, inspect it, then run it:

```sh
curl -fL https://github.com/tinkrtailor/cleared-cli/releases/download/v0.2.0/install.sh -o install.sh
less install.sh
sh install.sh --version 0.2.0
export PATH="$HOME/.local/bin:$PATH"
cleared --version
cleared --help
```

The installer selects your platform, verifies the archive against the release's
`SHA256SUMS`, and installs `cleared` to `~/.local/bin`. It does not use `sudo` or edit
shell startup files. Use `--install-dir DIR` to choose another destination. It requires
`curl`, standard shell/archive tools, and either `sha256sum` or `shasum`.

### Platforms

The four binary targets and their v0.2.0 execution checks are:

| Platform | Architecture          | Target                       | Test environment                      |
| -------- | --------------------- | ---------------------------- | ------------------------------------- |
| macOS    | Apple Silicon / ARM64 | `aarch64-apple-darwin`       | Native Apple Silicon                  |
| macOS    | Intel / x86-64        | `x86_64-apple-darwin`        | Rosetta on Apple Silicon              |
| Linux    | ARM64                 | `aarch64-unknown-linux-musl` | QEMU emulation                        |
| Linux    | x86-64                | `x86_64-unknown-linux-musl`  | Native x86-64                         |

These are test environments, not a claim of compatibility with every older OS version.
**The v0.2.0 macOS binaries are unsigned and not notarized.** macOS may block execution
or require approval in Privacy & Security; installation is not guaranteed to be seamless.

For manual installation, select `cleared-v0.2.0-TARGET.tar.gz` from the release and
verify it using `SHA256SUMS`. Each archive includes the executable,
`DISTRIBUTION-TERMS.txt`, and `THIRD-PARTY-NOTICES.md`.

The historical [v0.1.0 release](https://github.com/tinkrtailor/cleared-cli/releases/tag/v0.1.0)
remains available for API-key-only usage.

## Set up authentication

### Interactive browser login

With an OAuth-enabled Cleared API and web deployment, sign in through your browser:

```sh
cleared login
cleared auth status
cleared auth status --validate
cleared logout
```

Login opens the Cleared consent page using your Cleared/Privy browser session and
waits for a `127.0.0.1` loopback callback. It uses authorization code with S256 PKCE
and no client secret. To open the URL yourself:

```sh
cleared login --no-browser
```

Open the printed URL on the same machine as the CLI so the callback can complete.
For a remote/headless host where that is not possible, use API-key authentication.

Default consent is read-only: `invoice:read`, `customer:read`, and `dashboard:read`.
Request a comma-separated scope set when you need different access, for example:

```sh
cleared login --scope invoice:read,invoice:write,customer:read,dashboard:read
```

OAuth credentials are stored only in macOS Keychain or Linux Secret Service,
namespaced by the exact API and web origins. Linux login requires an available,
unlocked Secret Service on the user's D-Bus session. If the secure store is
unavailable, login fails closed; there is no plaintext-file fallback.

Access tokens last at most 15 minutes and refresh through a rotating session with
an absolute maximum lifetime of 30 days. Safe reads may refresh and retry once
after a 401; mutations are never automatically replayed after dispatch.
`auth status` is local and redacted by default. `auth status --validate` contacts
the server and may refresh the session. Logout attempts remote revocation and
always attempts local removal; check its result for failures.

### API keys for automation

1. Sign in to the [Cleared web app](https://clearedpay.xyz) and open **Settings → API Keys**.
2. Create an API key with the scopes you need. Save its key ID and secret when shown.
3. For invoice issuance or cancellation, complete the signing-session activation in
   the web app. If write access is pending, select **Activate Now** and complete the
   signing prompt. Issuance also requires a completed business profile and an existing customer.
4. Supply `CLEARED_KEY_ID` and `CLEARED_API_KEY` through your environment or secret manager.

For an interactive Bash session, these prompts avoid putting credentials into shell history:

```bash
read -r -p 'Cleared key ID: ' CLEARED_KEY_ID
read -r -s -p 'Cleared API secret: ' CLEARED_API_KEY
printf '\n'
export CLEARED_KEY_ID CLEARED_API_KEY
```

A complete `CLEARED_KEY_ID` / `CLEARED_API_KEY` environment pair takes precedence
over stored OAuth credentials. A partial pair is an error; the CLI does not fall
back to another identity. Unset both variables to use your browser-login identity.
`cleared logout` does not change API keys or these environment variables.

The CLI does not create API keys or activate signing sessions. It accepts no
secret command-line arguments and does not load `.env` files.

### Scopes and signing authority

Choose scopes for the operations you intend to use:

| Operation                                                  | Scope                                 |
| ---------------------------------------------------------- | ------------------------------------- |
| Get/list customers; resolve a customer by name or email    | `customer:read`                       |
| Create/update customers                                    | `customer:write`                      |
| Send/cancel invoices                                       | `invoice:write`                       |
| Read/list invoices, retrieve payment links, check payments | `invoice:read`                        |
| View dashboard                                             | `dashboard:read`                      |
| Read protocol information                                  | None; public and sends no credentials |

Select `customer:read` as well when an invoice or payment command needs customer
resolution. The web app pairs write scopes with their corresponding read scopes.
Invoice read-scope enforcement can vary by API route; request the intended scope above.

OAuth login grants API identity and consented scopes, not signing authority.
Invoice issuance and cancellation additionally require `invoice:write` and an
active, unexpired server signing session for the correct chain. Activate that
session in web Settings. Login can succeed while invoice writes return
`NO_ACTIVE_SIGNER`. Issuance also requires a completed business profile and an
existing customer.

## Examples

Read public protocol information without credentials:

```sh
cleared protocol info --chain base
```

List customers and invoices, or request JSON output:

```sh
cleared customers list --limit 20
cleared invoices list --status issued --limit 20
cleared --json dashboard
```

To issue an invoice, use an existing customer's UUID and a fresh idempotency key that
you generate and retain for that business operation. Replace the example values:

```sh
cleared invoices send \
  --customer 'CUSTOMER_UUID' \
  --amount 500.00 \
  --description 'Consulting services' \
  --idempotency-key 'YOUR_RETAINED_UNIQUE_OPERATION_KEY' \
  --due-days 30
```

Add `--send-email` if you want Cleared to email the invoice link. Amounts are USDC
decimals with up to six fractional digits. Due days must be between 1 and 3650.

Use an invoice ID or formatted invoice number returned by Cleared in place of `ID`:

```sh
cleared invoices get ID
cleared invoices pay-link ID
cleared payments check --invoice ID
cleared invoices cancel ID
```

Cancellation changes the invoice. Only `Issued` invoices can become `Canceled`;
`Paid` is terminal. The CLI checks payments but does not pay invoices.
Run any command with `--help` for its options.

## Automation and recovery

`--json` emits one JSON object per completed invocation, including errors. `--help`
and `--version` print informational text instead. Lists return one page; follow the
returned cursor for more results and inspect partial/coverage metadata.

| Exit code | Meaning                                                          |
| --------- | ---------------------------------------------------------------- |
| 0         | Complete                                                         |
| 1         | Unexpected local failure                                         |
| 2         | Invalid command, input, or configuration                         |
| 3         | Authentication, authorization, scope, or signing-session failure |
| 4         | Not found or ambiguous lookup                                    |
| 5         | Rejected request or failed read                                  |
| 6         | Mutation accepted but still pending                              |
| 7         | Mutation outcome uncertain; reconciliation required              |
| 8         | Partial result                                                   |

Retain the invoice issuance idempotency key and returned `meta.recovery.input`.
After pending or uncertain issuance, use the same key and exact recovery input;
never create a new key merely because a response was lost.
`IDEMPOTENCY_SUBMISSION_UNCERTAIN` requires reconciliation/support. A failed payment-link
lookup after issuance does not mean you should issue again.

After pending or uncertain cancellation, check `cleared invoices get ID` rather than
automatically resubmitting. A read of `Issued` does not prove an in-flight cancellation
failed. The CLI performs no automatic mutation retry or wait loop.

## Configuration

| Environment variable | Default                      |
| -------------------- | ---------------------------- |
| `CLEARED_API_URL`    | `https://api.clearedpay.xyz` |
| `CLEARED_PAY_URL`    | `https://clearedpay.xyz`     |
| `CLEARED_WEB_URL`    | Pay origin (normally `https://clearedpay.xyz`) |

Non-secret flags `--api-url`, `--pay-url`, `--web-url`, and `--timeout` (1–600 seconds) are also
available. Remote API endpoints require HTTPS. When a custom API returns relative
payment links, explicitly configure its matching pay URL. For browser login to a
custom deployment, configure its matching web origin with `--web-url` or
`CLEARED_WEB_URL`.

## Binary terms

Copyright Cleared. Permission is granted to download and use this binary to access Cleared. No rights to the private source code are granted. Provided AS IS, without warranty.

Third-party notices accompany each release archive.
