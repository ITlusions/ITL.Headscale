# ITL.Headscale documentation index

Documents are grouped into numbered folders. Each file is named `x.y-title.md`, where `x` is the folder number. Start with **1. Architecture**.

## 1. Architecture

- [1.1 Overview](1-architecture/1.1-overview.md): what this replaces, why Headscale over raw WireGuard, how it fits with `ITL.Headscale.UI` and `ITL.Keycloak.Tenants`.
- [1.2 Deployment](1-architecture/1.2-deployment.md): how to deploy, the bootstrap order, verifying it's healthy.

## 2. Authentication

- [2.1 OIDC setup](2-authentication/2.1-oidc-setup.md): the Keycloak client, `allowed_groups`, and the real bugs hit getting login to actually work.
- [2.2 Keycloak claims gotchas](2-authentication/2.2-keycloak-claims-gotchas.md): the `KeycloakOIDCClient` CRD's empty-by-default mappers, group-path formatting, and the exact failure sequence - written so the next person (or the next app) doesn't rediscover this the hard way.

## 3. Network

- [3.1 Subnet router](3-network/3.1-subnet-router.md): the in-cluster Tailscale client that makes cluster resources and the LAN reachable, not just the Headscale server itself. The Talos-specific fixes it needed.
- [3.2 ACL policy](3-network/3.2-acl-policy.md): the current (minimal, explicit allow-all) policy, why it exists, what file-mode vs database-mode means.

## 4. Operations

- [4.1 Troubleshooting](4-operations/4.1-troubleshooting.md): the real failures hit building this - config-checksum rollout gaps, shell-less container images, MSYS path mangling, dual-domain cert handling.
- [4.2 DERP relay follow-up](4-operations/4.2-derp-relay-followup.md): why a real device stays relayed via Tailscale's Amsterdam DERP, what self-hosting would and wouldn't fix, scoped as a deferred follow-up.

## Related repos

- [`ITL.Headscale.UI`](https://github.com/ITlusions/ITL.Headscale.UI): Headplane, the admin web UI - separate chart, separate release.
- [`ITL.Keycloak.Tenants`](https://github.com/ITlusions/ITL.Keycloak.Tenants): the Keycloak server itself, and `files/realm-config/itlusions-shared.yaml` - the realm-wide groups and the bootstrap automation this cluster's `KeycloakOIDCClient` CRD usage depends on.
- [`ITL.Wireguard`](https://github.com/ITlusions/ITL.Wireguard): the design this replaced - kept for reference, not deployed.
- [`ITL.K8s`](https://github.com/ITlusions/ITL.K8s): the main cluster repo - `docs/3-authentication/3.4-personas-and-roles.md` is where the `/vpn-users` group and the wider persona model are documented.
