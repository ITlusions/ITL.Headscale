# ITL.Headscale

Self-hosted [Headscale](https://headscale.net/) control server for the ITL Talos cluster ([`ITlusions/ITL.K8s`](https://github.com/ITlusions/ITL.K8s)) - a Tailscale-compatible mesh VPN coordination server (open-source reimplementation, not Tailscale Inc.'s own SaaS), gated by Keycloak OIDC so every device join requires a real login, not just a static key. Pairs with [`ITL.Headscale.UI`](https://github.com/ITlusions/ITL.Headscale.UI) (Headplane) for the admin web UI - separate chart/release, deployed independently.

**Replaces [`ITL.Wireguard`](https://github.com/ITlusions/ITL.Wireguard)'s design**, not an addition to it: that chart gave remote users a single gateway with static, hand-distributed peer configs - no connection-time authentication, no revocation beyond manually editing the peer list. Headscale moves that to a real mesh model with OIDC-backed device authorization, matching the `ITL.AuthGateway` pattern used elsewhere in this org (the "can we add OIDC to WireGuard" question - stock WireGuard structurally can't, Headscale is the actual answer).

**Status: deployed and live** at `https://headscale.itlusions.com` (and `.nl`), verified end-to-end with a real device over both LAN and cellular. Built 2026-10-23, confirmed working 2026-10-24.

## Why SQLite, not Postgres

Headscale's own docs now call Postgres "highly discouraged... only supported for legacy reasons" - reversed from older guidance that recommended it. SQLite on a persistent volume is current upstream's recommendation, so that's what this chart uses - same `/var/lib/headscale` persistence pattern as `ITL.Wireguard`'s `/config` PVC.

## Storage class

`ha-gen-lh`, not `ha-secure-lh` - `ha-secure-lh`'s StorageClass has a broken Longhorn parameter on this cluster (`invalid ReplicaSoftAntiAffinity setting: true`, confirmed 2026-10-23 while deploying `ITL.Wireguard`). Unrelated to this chart, but affects any PVC using that class - worth fixing in `ITL.Longhorn` separately.

## Public exposure, deliberately different from ITL.Wireguard

Headscale's client protocol is HTTP(S) (long-lived connections, HTTP/2) - unlike raw WireGuard UDP, it *can* sit behind a reverse proxy. This chart fronts it with the **public** Traefik instance (`headscale.itlusions.com`, cert-manager/Let's Encrypt, same pattern as the website). Headscale's own docs note they don't officially test/support reverse proxies though - watch for connection-drop issues under sustained load, and be ready to front it directly with a `LoadBalancer` Service instead if that happens.

## Two separate Keycloak clients, not one

This chart's `KeycloakOIDCClient` (`keycloak.clientId: headscale`) gates `tailscale up --login-server=...` - i.e. whether a device/user can join the tailnet at all. `ITL.Headscale.UI`'s own `KeycloakOIDCClient` (`headplane`) is a completely separate client gating admin-UI login. Same Keycloak realm, different applications, different blast radius if one secret leaks.

## Open items, not yet resolved

- `trusted_proxies` is set to the whole pod CIDR (`10.244.0.0/16`) as a placeholder - scope this down once the actual Traefik pod's source IP behavior is confirmed.
- Headscale's reverse-proxy support is explicitly "not officially tested" per upstream docs - this is a real risk to watch for, not a resolved question.
- **DERP relies on Tailscale's own public relay servers** (`controlplane.tailscale.com/derpmap/default`) rather than self-hosting one. Confirmed live 2026-10-24: a real device on cellular data stayed relayed through Tailscale's Amsterdam DERP (`derp-14`/`ams`) even after removing the in-cluster gateway's own NAT layer (`subnetRouter.hostNetwork: true`, see below) - almost certainly the phone's own carrier-grade NAT (common on cellular, usually symmetric/restrictive, very hard to traverse from either side). Self-hosting a DERP server **would not fix that** - it doesn't turn a relayed connection into a direct one, NAT traversal either works or it doesn't regardless of whose relay picks up the slack. What it *would* do: remove the extra hop to Tailscale's external Amsterdam relay for whatever traffic stays relayed anyway, and keep that traffic inside infrastructure we control instead of a third party's. Concretely, this chart's `derp.server` block (currently `enabled: false` in `templates/configmap.yaml`) needs:
  - `enabled: true`, a generated DERP server private key, `stun_listen_addr` set
  - The real public IP/hostname in the `ipv4`/`ipv6`/region fields
  - **A new UDP-exposed port** (STUN, typically `3478`) - same class of problem `ITL.Wireguard` hit originally: raw UDP can't go through Traefik's HTTP(S) proxying, needs its own `LoadBalancer` Service via MetalLB and the usual external-reachability/router considerations
  
  Scoped as a real follow-up, not started - deliberately deferred 2026-10-24 given the hour and that it doesn't solve the actual NAT problem that prompted the investigation.
- `oidc.allowedGroups` is empty by default - decide and set a real Keycloak group before this goes live, otherwise every valid Keycloak user in the realm can join the tailnet.

## Deploy

```bash
cd itl.headscale
# the oidc client secret (headscale-oidc-secret, key client-secret) must
# exist before this will actually authenticate anyone - either let the
# KeycloakOIDCClient CR create it (keycloak.enabled: true) or create it
# by hand first.
helm upgrade --install headscale . -n itl-headscale --create-namespace
```

## After deploying: generate Headplane's API key

`ITL.Headscale.UI` needs an API key to manage this server - generate it only after this chart is running:

```bash
kubectl exec -n itl-headscale deploy/headscale -- headscale apikeys create
# then create the Secret ITL.Headscale.UI expects:
kubectl create secret generic headplane-headscale-api-key \
  -n itl-headscale-ui --from-literal=api-key=<the key printed above>
```

## Add a user/device

With OIDC, there's no per-device config file to generate and hand out like `ITL.Wireguard` had - the user runs the Tailscale client themselves and logs in:

```bash
tailscale up --login-server=https://headscale.itlusions.com
```

They'll be sent through Keycloak. Whether they're allowed to actually join is controlled by `oidc.allowedGroups`/`allowed_users`/`allowed_domains` in `values.yaml`'s `oidc` block.

## Verify

```bash
kubectl get pods -n itl-headscale
kubectl logs -n itl-headscale deploy/headscale
curl -s https://headscale.itlusions.com/health
```
