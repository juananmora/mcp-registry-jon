# MCP Registry - Instancia local

Instancia local del [MCP Registry](https://github.com/modelcontextprotocol/registry) sembrada con un catalogo controlado (solo los MCPs que queremos exponer en este entorno).

## Stack

- **registry**: `ghcr.io/modelcontextprotocol/registry:1.7.8` (version estable verificada localmente)
- **postgres**: `postgres:16-alpine` (datos en volumen Docker, no persistente fuera del contenedor)
- Puerto API: `http://localhost:8080`
- Puerto Postgres (debug): `localhost:5432`

## Estructura

```
.
├── docker-compose.yml      # stack registry + postgres
├── .env.example            # plantilla de configuracion (copiar a .env)
├── data/
│   └── seed.json           # catalogo de MCPs cargado al primer arranque
└── CLAUDE.md
```

## Arranque

```powershell
# Primera vez (o tras editar seed.json para regenerar el catalogo)
docker compose down -v       # -v borra el volumen de postgres -> recarga seed
docker compose up -d

# Ver progreso
docker compose logs -f registry
```

El log debe terminar con:

```
Import completed successfully: all N servers created
HTTP server starting on :8080
```

> El seed **solo se aplica cuando la BD esta vacia**. Si modificas `data/seed.json` y haces `docker compose restart`, los cambios **no se aplican**. Hay que `docker compose down -v` para forzar la recarga.

## Endpoints utiles

| Endpoint | Uso |
|---|---|
| `GET /v0/health` | health check + github client id en uso |
| `GET /v0/ping` | liveness |
| `GET /v0/servers?limit=100` | listar servers (paginado, max 100 por pagina) |
| `GET /v0/servers?search=texto` | buscar por nombre |
| `GET /v0/servers/{name}/versions` | lista de versiones de un server |
| `GET /v0/servers/{name}/versions/{version}` | detalle de una version (la "receta" que consume un cliente) |
| `GET /v0/version` | version del propio registry |
| `POST /v0/auth/none` | obtener token anonimo (si `ENABLE_ANONYMOUS_AUTH=true`) |
| `POST /v0/publish` | publicar un server (requiere `Authorization: Bearer <token>`) |
| `GET /openapi.json`, `GET /docs` | spec OpenAPI completo y Swagger UI |

> **Importante:** el nombre del server contiene `/` (ej. `io.github.github/github-mcp-server`).
> En la URL ese `/` debe ir **URL-encoded como `%2F`**. No existe `GET /v0/servers/{name}` — usa `/versions` o `/versions/{version}`.

Ejemplos:

```bash
# Contar servers
curl -s http://localhost:8080/v0/servers?limit=100 | grep -oE '"name":"io\.[^"]+"' | sort -u | wc -l

# Versiones disponibles de github
curl -s "http://localhost:8080/v0/servers/io.github.github%2Fgithub-mcp-server/versions"

# Receta completa de una version concreta
curl -s "http://localhost:8080/v0/servers/io.github.github%2Fgithub-mcp-server/versions/0.19.1"
```

## Catalogo actual

Definido en `data/seed.json`. Formato: **array JSON plano** con objetos `ServerJSON` (no envolver en `{"servers":[...]}` — eso es el formato de salida del endpoint, no de entrada para el seed).

Actualmente sembrado con:
- `io.github.github/github-mcp-server-remote` v0.33.0 (remoto: `https://copilot-api.bbva.ghe.com/mcp`, transport `streamable-http`)
- `io.github.microsoft/playwright-mcp` v0.0.75 (npm: `@playwright/mcp`)

## Anadir / quitar MCPs

### Opcion A: editar `data/seed.json` y reiniciar limpio (recomendado)

1. Edita `data/seed.json` anadiendo otro objeto al array.
2. `docker compose down -v && docker compose up -d`.

Esquema minimo de un server:

```json
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-10-17/server.schema.json",
  "name": "io.github.OWNER/REPO",
  "description": "Una linea",
  "repository": { "url": "https://github.com/OWNER/REPO", "source": "github" },
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "nombre-paquete",
      "version": "1.0.0",
      "transport": { "type": "stdio" }
    }
  ]
}
```

`registryType` admite `npm`, `pypi`, `oci`, `nuget`. Para servidores remotos (HTTP/SSE) usa `remotes` en vez de `packages`.

> **Tip:** copia el JSON canonico de cualquier MCP publico:
> ```
> curl -s "https://registry.modelcontextprotocol.io/v0/servers?search=NOMBRE" | jq '.servers[] | .server'
> ```
> Despega cada `server` del envoltorio y pegalo en el array.

### Opcion B: publicar via API en caliente

Solo sirve para namespace `io.modelcontextprotocol.anonymous/*` (los `io.github.*/*` requieren OAuth GitHub real).

```bash
# 1) Token anonimo
TOKEN=$(curl -s -X POST http://localhost:8080/v0/auth/none \
  -H "Content-Type: application/json" -d '{}' \
  | grep -oE '"registry_token":"[^"]+"' | sed 's/.*"registry_token":"//;s/"$//')

# 2) Publicar (body = ServerJSON pelado, sin envoltorio)
curl -X POST http://localhost:8080/v0/publish \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @mi-server.json
```

## Configuracion (.env)

`docker-compose.yml` usa la forma `${VAR:-default}` con valores de demo. Para sobreescribirlos, copia `.env.example` a `.env`. Variables relevantes:

| Variable | Default | Uso |
|---|---|---|
| `MCP_REGISTRY_SEED_FROM` | `/data/seed.json` | ruta o URL del catalogo inicial |
| `MCP_REGISTRY_ENABLE_REGISTRY_VALIDATION` | `false` | si `true` valida que cada paquete exista en npm/PyPI/OCI (necesita salida a internet) |
| `MCP_REGISTRY_ENABLE_ANONYMOUS_AUTH` | `true` | habilita `/v0/auth/none` |
| `MCP_REGISTRY_JWT_PRIVATE_KEY` | demo | rotar con `openssl rand -hex 32` antes de exponer |
| `MCP_REGISTRY_GITHUB_CLIENT_SECRET` | demo | rotar o vaciar — el default es publico del repo upstream |
| `MCP_REGISTRY_OIDC_ENABLED` | `true` | desactivar (`false`) si no usas Google/OIDC |

## Mantenimiento

```powershell
# Reiniciar solo el registry (cambios de env vars o imagen)
docker compose up -d --force-recreate registry

# Pull de imagen nueva
docker compose pull && docker compose up -d

# Resetear todo (incluye datos)
docker compose down -v

# Inspeccionar Postgres
docker exec -it postgres psql -U mcpregistry -d mcp-registry -c "SELECT name, version FROM servers;"

# Logs
docker compose logs -f registry
docker compose logs -f postgres
```

## Limitaciones conocidas

- El seed **no es idempotente con cambios**: para reaplicar `seed.json` modificado hay que `down -v`.
- `limit` de `/v0/servers` esta topado a 100 — usar `cursor` (`metadata.nextCursor`) para paginar.
- El token anonimo solo puede publicar en `io.modelcontextprotocol.anonymous/*`. Otros namespaces requieren OAuth real.
- Si activas `ENABLE_REGISTRY_VALIDATION=true`, el seeder llama a npm/PyPI/OCI para cada paquete — falla si no hay internet o si el paquete no existe (no usar para datos ficticios).
