# Postiz on Coolify (dev-dmytro)

Upstream: https://github.com/gitroomhq/postiz-docker-compose
`docker-compose.yaml` and `dynamicconfig/` are copied verbatim from upstream;
only the adaptations below are ours.

## Adaptations from upstream

| Upstream | Here | Why |
|---|---|---|
| `ports: "4007:5000"` | `expose: "5000"` + Traefik labels | the platform Traefik is the only ingress; no host ports are published |
| `ports: 127.0.0.1:7233` (temporal) | `expose: "7233"` | no host port publishing |
| `ports: 127.0.0.1:8080` (temporal-ui) | `expose: "8080"` | 8080 is the Traefik dashboard port on this host |
| - | external `coolify` network | the managed Traefik network; the stack must join it for the docker provider to see it |
| `container_name:` on every service | removed | the platform prefixes container names; fixed names collide and break redeploys |
| hardcoded URLs / secrets in `environment:` | `${VAR}` from platform env | public URL and secrets come from the deployment, not from the file |

## Traefik routing

Postiz is a compose stack, so the router is declared through container labels
(the same pattern as the other applications on this host):

    traefik.http.routers.postiz.rule=Host(`${PUBLIC_HOST}`)
    traefik.http.routers.postiz.entrypoints=${TRAEFIK_ENTRYPOINT}
    traefik.http.routers.postiz.tls=true
    traefik.http.services.postiz.loadbalancer.server.port=5000

The platform escapes `${VAR}` inside labels to `$${VAR}` by default, so
`is_container_label_escape_enabled` is **false** on the application - otherwise
Traefik matches the literal string `${PUBLIC_HOST}`.

## Required environment

| Variable | Purpose |
|---|---|
| `PUBLIC_HOST` | public hostname, e.g. `postiz.idstar.de` |
| `PUBLIC_URL` | `https://<PUBLIC_HOST>` - used for OAuth redirect URIs |
| `TRAEFIK_NETWORK` | `coolify` |
| `TRAEFIK_ENTRYPOINT` | `websecure` |
| `TRAEFIK_CERTRESOLVER` | `letsencrypt` |
| `JWT_SECRET` | session signing secret |
| `POSTGRES_PASSWORD` | Postiz database password |
| `TEMPORAL_POSTGRES_PASSWORD` | Temporal database password |

Social provider keys (X, LinkedIn, Reddit, YouTube, TikTok, ...) and
`OPENAI_API_KEY` stay empty until a provider is connected.

## Notes

- The official image serves frontend and backend from one container on port 5000.
- Social OAuth providers require an HTTPS redirect URI, so `PUBLIC_URL` must be
  an `https://` URL - otherwise Postiz wraps the redirect through
  `redirectmeto.com`.
- TikTok and a few providers fetch media by URL, so `/uploads` must be reachable
  from the public internet over HTTPS (it is, via Traefik).
