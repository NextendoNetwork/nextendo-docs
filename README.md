<h1 align="center">Nextendo Network Documentation</h1>

<p align="center">
  <b>How the Nextendo Network server stack fits together, and how to run your own.</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-PolyForm%20Shield%201.0.0-orange" alt="License">
</p>

---

**Nextendo Network** is a community-run replacement for a retired console online service. It lets a
compatible client (the Nextendo emulator, a Ryujinx fork, or a real console pointed at it via DNS)
reach matchmaking, friends, presence, and cloud saves again, using a small set of self-hostable Go
services that speak the same protocols the retail servers did.

This repository is the **documentation**: the map of the whole system and a guide to deploying it.
The code lives in the per-component repositories.

## Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)**: the components, how they connect, and what happens when a
  game goes online (the full request flow).
- **[DEPLOYMENT.md](DEPLOYMENT.md)**: running the stack yourself: prerequisites, per-service
  configuration, DNS/TLS setup, ports, and an example `docker-compose`.

## The repositories

| Repository | Role |
| ---------- | ---- |
| **nextendo-nex** | The from-scratch NEX / PRUDP protocol core the game servers are built on. |
| **mario-kart-8-deluxe**, **splatoon-2**, **super-smash-bros-ultimate** | Per-game NEX servers (auth + secure). |
| **nextendo-account** | Accounts, friends, presence, BCAT, sessions, token signing. |
| **nextendo-nncs** | NAT-check (NCS) responder so peer-to-peer can establish. |
| **nx-dauth** | Device/application authentication chain (self-signed). |
| **sni-router** | TLS SNI passthrough so multiple auth servers share `:443`. |
| **nx-scsi** | Cloud saves (optional). |
| **nextendo-dashboard** | Live monitoring of the game servers (optional). |
| **nextendo-site** | The public website (account UI). |

## Design principles

- **No baked-in infrastructure.** Every address, port, and secret is supplied through environment
  variables or files at runtime. Nothing in these repositories points at any specific deployment.
- **No Nintendo code or data.** These are independent reimplementations of publicly-observable
  protocols. You must provide your own legally-dumped games, keys, and system files, exactly as with
  the upstream emulator.
- **Self-hostable.** Everything here is meant to run on your own machine or server.

## License

Documentation and code are released under the **[PolyForm Shield License 1.0.0](LICENSE.md)**, a
source-available license: read, use, modify, and self-host, but do not use it to provide a product
that competes with Nextendo Network.

> Nextendo Network is an independent, non-commercial project. It is not affiliated with, endorsed by,
> or associated with Nintendo. All game and product names are trademarks of their respective owners.
