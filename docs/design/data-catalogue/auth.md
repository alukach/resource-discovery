# Access Control

The STAC API of the Data Catalogue (eoAPI) is protected by [STAC Auth Proxy](https://github.com/developmentseed/stac-auth-proxy), which sits in front of the STAC API and enforces authentication and authorization for every request. The proxy validates OpenID Connect (OIDC) tokens issued by the [IAM Building Block](https://eoepca.readthedocs.io/projects/iam/) (Keycloak) and applies access policies by injecting [CQL2](https://docs.ogc.org/is/21-065r2/21-065r2.html) filters into requests before they reach the STAC API.

This approach was designed in [EOEPCA/resource-discovery#203](https://github.com/EOEPCA/resource-discovery/issues/203) and implemented in [EOEPCA/eoepca-plus#118](https://github.com/EOEPCA/eoepca-plus/pull/118).

## Architecture

```
Client (e.g. STAC Manager, pystac-client, curl)
  │  Authorization: Bearer <OIDC token>   (optional for public reads)
  ▼
STAC Auth Proxy ── validates JWT against IAM (Keycloak)
  │                derives a CQL2 filter from the token claims
  ▼
eoAPI STAC API (stac-fastapi-pgstac) ── applies the filter to every query
```

The proxy enforces policy through two mechanisms:

- **Read requests** (`GET`, `HEAD`, `OPTIONS`, and `POST` to `/search`): the proxy appends a CQL2 filter to the query, so the STAC API only returns collections and items the caller may see.
- **Write requests** (`POST`, `PUT`, `PATCH`, `DELETE` on collections and items): the proxy evaluates the same policy against the target collection and rejects the request if the caller lacks write access.

Because the filter is generated as CQL2-JSON (a structured expression, not concatenated text), values taken from the token can never alter the shape of the query — there is no injection path from token claims into the filter.

A few endpoints are public and bypass authentication entirely: the landing page (`/`), the API description (`/api`, `/api.html`), `/conformance`, and the health check (`/healthz`).

## Policy model

Access is governed by an opinionated naming convention for collection IDs. A collection's ID prefix — the part before the first `.` — determines who can read and write it.

| Collection ID pattern        | Example                   | Read                                                       | Write                              |
| ---------------------------- | ------------------------- | ---------------------------------------------------------- | ---------------------------------- |
| No prefix (no `.` in the ID) | `sentinel-2-l2a`          | Everyone, including anonymous users                        | `stac_editor` role only            |
| `<username>.<collection>`    | `alice.my-experiments`    | User `alice`                                               | User `alice`                       |
| `<group>.<collection>`       | `pn56su-dss-0034.landsat` | Members of group `pn56su-dss-0034` or `pn56su-dss-0034-ro` | Members of group `pn56su-dss-0034` |

### Public collections

Any collection whose ID contains no `.` is public: anonymous and authenticated users alike can read it and its items. Only callers holding the [`stac_editor` role](#service-accounts-the-stac_editor-role) can create or modify public collections.

### User collections

An authenticated user can read and write any collection prefixed with their username (the `preferred_username` claim of their token) followed by a `.`. Users may create as many collections under their own prefix as they like.

### Group collections

Group permissions derive from the `groups` claim of the user's token. The policy recognizes group names of the form `/dss/<group-id>`, where `<group-id>` must contain `-dss-`:

- `/dss/pn56su-dss-0034` grants **read and write** access to collections prefixed `pn56su-dss-0034.`
- `/dss/pn56su-dss-0034-ro` grants **read-only** access to the same collections
- `/dss/pn56su-dss-0034-mgr` is ignored — the `-mgr` suffix denotes a storage-management role, which carries no data access rights

Groups that lack the `/dss/` prefix or the `-dss-` infix are ignored.

!!! warning "Collection and item permissions are coupled"
    Write access to a collection's items implies write access to the collection itself. A user who can add items to `pn56su-dss-0034.landsat` can also edit or delete that collection's metadata — and can create new collections under any prefix they hold.

### Service accounts: the `stac_editor` role

Automated services such as the [Registration Harvester](https://eoepca.readthedocs.io/projects/resource-registration/) authenticate with client-credentials tokens, which carry no username or groups. For these callers the policy supports a write bypass: a token grants **unrestricted, catalog-wide write access** when both of the following hold:

1. The token was issued for one of the trusted client IDs, verified via the token's `azp` claim, and
2. The token carries the `stac_editor` client role under `resource_access.<client>.roles`.

The trusted clients and the role name are set via the proxy's `STAC_EDITOR_CLIENT_IDS` and `STAC_EDITOR_ROLE` environment variables; in the EOEPCA+ demo cluster the trusted clients are `eoapi` and `registration-harvester`. The role itself is assigned in Keycloak, so granting or revoking catalog-wide write access is an IAM operation — no redeployment involved. Each use of the bypass is audit-logged with the token's `azp`, `sub`, and `jti` claims.

!!! danger "Only trust confidential clients"
    `STAC_EDITOR_CLIENT_IDS` must only list confidential clients whose role assignment is controlled by operators. Never list a public client where users can self-register, as the role check anchors on the token's issuing client.

### Default deny

A request that matches no policy is denied: an anonymous write, or an authenticated request against a collection outside the caller's username and group prefixes, receives a filter that matches nothing.

## Implementation

The policy is implemented as custom [filter classes](https://github.com/developmentseed/stac-auth-proxy#filters) for STAC Auth Proxy, defined in [`eoepca_filters.py`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/stac-auth-proxy/eoepca_filters.py) in the `eoepca-plus` deployment repository and wired into the proxy via its `COLLECTIONS_FILTER_CLS` and `ITEMS_FILTER_CLS` settings in the [Helm values](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/values/values-eoapi.yaml). The two filter classes share one policy implementation; they differ only in the property they filter on (`id` for collections, `collection` for items).

A Kustomize `configMapGenerator` packages the policy file into a ConfigMap, which is mounted into the proxy container at runtime. Policy logic can therefore be changed by editing a single Python file in the deployment repository — no proxy image rebuild or chart upgrade is required. ArgoCD applies the updated ConfigMap, and the proxy pods load the new policy on their next restart.

The policy logic is covered by a unit test suite ([`test_eoepca_filters.py`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/stac-auth-proxy/test_eoepca_filters.py)) that runs in CI on every change to the deployment repository. To run it locally:

```bash
uv run --with pytest --with pytest-asyncio --with cql2 \
  pytest argocd/eoepca/data-access/parts/stac-auth-proxy/test_eoepca_filters.py
```

!!! note "Version requirement"
    The custom filter classes require STAC Auth Proxy `v1.0.0` or later.

## Client behavior

Because read responses depend on the caller's identity, clients should send their OIDC token on **every** request to the STAC API — not only on writes. An unauthenticated `GET /collections` returns only public collections; the same request with a `Authorization: Bearer <token>` header additionally returns the caller's user and group collections.

[STAC Manager](https://github.com/developmentseed/stac-manager) (the basis of the [Resource Administration UI](../resource-admin-ui/design.md)) follows this pattern as of `v1.0.0`: when a user is logged in, it attaches their token to all STAC API requests — collection listings, item searches, and transactions alike — so users see their private collections throughout the UI.

Command-line and notebook users can do the same, e.g. with `pystac-client`:

```python
from pystac_client import Client

client = Client.open(
    "https://eoapi.develop.eoepca.org/stac",
    headers={"Authorization": f"Bearer {token}"},
)
```

## References

- [EOEPCA/resource-discovery#203](https://github.com/EOEPCA/resource-discovery/issues/203) — design discussion of the policy model
- [EOEPCA/eoepca-plus#118](https://github.com/EOEPCA/eoepca-plus/pull/118) — implementation of the CQL2 filter logic
- [STAC Auth Proxy](https://github.com/developmentseed/stac-auth-proxy) — the proxy enforcing the policies
- [stac-manager#71](https://github.com/developmentseed/stac-manager/pull/71) — STAC Manager sending credentials on all requests
