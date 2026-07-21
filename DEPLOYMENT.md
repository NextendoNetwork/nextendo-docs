# Deployment

This guide walks through running your own Nextendo Network. It is deliberately generic — every value
below is a placeholder you replace with your own. **No real addresses or secrets appear anywhere in
these repositories.**

> ⚠️ You must supply your own legally-dumped games, keys, and system files. Nextendo ships none of
> Nintendo's code, keys, or data.

## Prerequisites

- **Go 1.23+** to build the services (the account server needs 1.21+).
- **Two public IP addresses**, or two addresses your clients resolve distinctly:
  - one for the main services (call it `SERVER_IP`),
  - one for the NAT-check second vantage point (call it `NAT_IP`).
  The NAT check compares what two different addresses observe, so they must be distinct.
- **A DNS you control for the clients** — either the emulator's built-in DNS redirect, or a real DNS
  server the console uses — to point the retail online hostnames at your `SERVER_IP` / `NAT_IP`.
- **A TLS certificate + CA** that the client trusts, valid for the redirected hostnames. On the
  emulator this is provided by its certificate-trust layer; a real console needs the CA installed.

## Addresses & ports (example)

| Service | Listens | Notes |
| ------- | ------- | ----- |
| `sni-router` | `:443` | Fronts all game auth servers; routes by SNI. |
| `mario-kart-8-deluxe` | auth via router, secure `:60003` | |
| `splatoon-2` | auth via router, secure `:60004` | |
| `super-smash-bros-ultimate` | auth via router, secure `:60005` | |
| `nextendo-account` | `:8080` (`/api/*` public, `/internal/*` gated) | |
| `nx-dauth` | `:443` (own hosts, via router or its own IP) | |
| `nextendo-nncs` | UDP probe ports on `SERVER_IP` **and** `NAT_IP` | |
| `nx-scsi` | `:443` (own hosts) | optional |
| `nextendo-dashboard` | `:8085` | optional |

Ports are examples; every service takes its bind address/port from the environment.

## Configuration

Each service repository ships an `example.env`. Copy it to `.env`, fill in real values, and **never
commit your real `.env`**. The values that must line up across services:

- **`NEXTENDO_SECRET` / signing key** — must be **identical** between `nextendo-account` and every
  game server, or the game servers will reject the account's `nx2.` tokens.
- **`NEXTENDO_INTERNAL_KEY`** — the shared key that lets a game server call the account server's
  `/internal/*` routes.
- **`NNCS_SERVER_IP`** — the public address the NAT responder reports back (your `SERVER_IP`).

Secrets (`NEXTENDO_SECRET`, `NEXTENDO_SIGN_KEY`, `NEXTENDO_INTERNAL_KEY`, `NEXTENDO_ADMIN_KEY`, SMTP
credentials, `DASH_TOKEN`) should be strong random values. The admin space stays closed until you set
`NEXTENDO_ADMIN_EMAILS`.

## Build & run a single service

```sh
git clone <repo-url> && cd <repo>
cp example.env .env      # edit .env
go build -o server .
./server
```

## DNS

Point the client's DNS at your servers. Conceptually:

```
*.<online-domains>            ->  SERVER_IP
<the second NAT-check host>   ->  NAT_IP
```

On the emulator, this is done for you: it redirects the online hostnames to the addresses you give it
via `NEXTENDO_SERVER_IP` and `NEXTENDO_NAT_IP`, and signs the account token with
`NEXTENDO_BAAS_SIGNING_KEY`. Set those three, point them at this deployment, and the client is done.

## TLS

The game auth servers and `nx-dauth`/`nx-scsi` expect TLS on `:443`. Generate a CA and per-host
certificates for the redirected hostnames, install the CA in the client's trust store (the emulator's
cert-trust layer handles this), and give each service its cert/key via `CERT_FILE` / `KEY_FILE`.
`sni-router` does **not** need certificates — it forwards the encrypted stream unopened.

## Example `docker-compose.yml`

Placeholders only — replace `SERVER_IP`, `NAT_IP`, and every `change-me`.

```yaml
services:
  router:
    build: ./sni-router
    ports: ["443:443"]
    environment:
      MK8_AUTH: "mk8:443"
      SSBU_AUTH: "ssbu:443"

  account:
    build: ./nextendo-account
    environment:
      NEXTENDO_BASE_URL: "https://your-domain.example"
      NEXTENDO_SECRET: "change-me"
      NEXTENDO_INTERNAL_KEY: "change-me"
      NEXTENDO_ADMIN_EMAILS: "you@example.com"

  mk8:
    build: ./mario-kart-8-deluxe
    environment:
      NEXTENDO_HOST: "0.0.0.0:60003"
      NEXTENDO_ACCOUNT_URL: "http://account:8080"
      NEXTENDO_SECRET: "change-me"          # same as account
      NEXTENDO_INTERNAL_KEY: "change-me"    # same as account
      NEXTENDO_SECURE_PASSWORD: "change-me"

  nncs:
    build: ./nextendo-nncs
    environment:
      NNCS_SERVER_IP: "SERVER_IP"
    network_mode: host   # needs the two distinct addresses

  # dauth, scsi, dashboard: add as needed
```

`splatoon-2` and `super-smash-bros-ultimate` follow the same shape as `mk8`. Their DataStore layer is
currently a stub, so their matchmaking is not complete out of the box — see each repo's README.

## Verifying it works

1. Start `account`, then the game servers (they need the account server for tokens).
2. Start `sni-router` and `nncs`.
3. Point the client at `SERVER_IP` / `NAT_IP`, launch a supported game, and go online.
4. Watch each service's log — every request is logged, so you can see exactly where a connection
   stops if something is misconfigured.

## Security checklist before exposing anything

- `NEXTENDO_ADMIN_EMAILS` set (admin is otherwise closed) and all secrets are strong + unique.
- `/internal/*` is **not** reachable from the internet — keep the account server's internal routes on
  a private network and rely on `NEXTENDO_INTERNAL_KEY` for off-network callers.
- Real `.env` files, certificates, and keys are never committed.
