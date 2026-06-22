# Frappe Handbook

A set of how-to guides for running **Frappe V15** across three environments. Each
guide stands on its own: read one top to bottom for a first-time setup, or jump to
the section whose heading names the task in front of you.

## Guides

1. [**Local Development**](01-local-development.md) — a development bench on Windows
   under WSL Ubuntu. Start here to build and test apps before they ship.
2. [**Production Deployment**](02-production-deployment.md) — a bare-metal install
   on Ubuntu 22.04 behind Cloudflare, where every subdomain is its own site under a
   single bench (DNS multitenancy).
3. [**Mobile Frontend Hosting**](03-mobile-frontend-hosting.md) — an Angular/Ionic
   single-page app served at `/mobile` on a bench, shipped as versioned tarballs
   from GitHub Releases.

## Conventions

Every guide follows these conventions.

- A `sudo` prefix means the command needs root. A command with no prefix runs as
  the current user — the `frappe` user in production, your own user locally.
- ⚠️ marks an operation that is security-sensitive or hard to undo. Read it before
  you run it.
- 💡 marks a tip or clarification.
- Verification closes each procedure: a sentence stating the expected outcome,
  followed by the command or observable result that confirms it.

## Placeholders

Substitute your own values for these throughout.

| Placeholder | Meaning | Example |
| ----------- | ------- | ------- |
| `~/frappe/frappe-bench` / `<bench>` | The bench root directory | `/home/frappe/frappe/frappe-bench` |
| `<frappe-app>` | A Frappe app name | `my_company_app` |
| `<angular-app>` | An Angular app name (inside the monorepo) | `mobile-app` |
| `<monorepo>` | The Angular monorepo name | `my-frontend` |
| `<github-org>` | Your GitHub organization or user | `acme-inc` |
| `example.com` | Your apex domain | — |
| `a.example.com`, `b.example.com`, … | Tenant subdomains | — |
| `<host>` | The subdomain a browser hits | `a.example.com` |
| `<server public IP>` | The server's public IPv4 address | — |
