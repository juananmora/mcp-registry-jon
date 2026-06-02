# MCP Registry en Kubernetes local (Rancher Desktop)

Esta carpeta contiene una variante local de los manifiestos para probar `mcp-registry` en un clúster de Rancher Desktop.

## Diferencias frente a `k8s/`

- No incluye `Ingress`.
- No define `storageClass`.
- PostgreSQL se ejecuta con un volumen `emptyDir`, suficiente para pruebas locales.
- El servicio del registry se expone como `NodePort` en el puerto `30080`.

## Archivos

- `00-namespace.yaml`: namespace `mcp-registry`.
- `01-registry-seed-configmap.yaml`: `seed.json` montado como volumen en `/data/seed.json`.
- `02-registry-configmap.yaml`: configuración no sensible del registry.
- `03-secrets.yaml`: credenciales y secretos del despliegue.
- `04-postgres-deployment.yaml`: PostgreSQL para entorno local.
- `05-postgres-service.yaml`: servicio interno para PostgreSQL.
- `06-registry-deployment.yaml`: despliegue del MCP Registry.
- `07-registry-service.yaml`: servicio `NodePort` para acceder al registry desde tu máquina.
- `kustomization.yaml`: archivo estándar de Kustomize para desplegar todo el conjunto.

## Catalogo actual

Servers sembrados en `01-registry-seed-configmap.yaml`:

- `io.github.github/github-mcp-server` v0.19.1 — local, paquete OCI `ghcr.io/github/github-mcp-server` (transport `stdio`, requiere `GITHUB_PERSONAL_ACCESS_TOKEN`).
- `io.github.github/github-mcp-server-remote` v0.19.1 — remoto hosted en `https://api.githubcopilot.com/mcp/` (transport `streamable-http`, sin Docker local).
- `io.github.microsoft/playwright-mcp` v0.0.75 — npm `@playwright/mcp` (transport `stdio`).

## Despliegue

```text
kubectl apply -k k8s-local
```

## Acceso

Con Rancher Desktop normalmente podrás acceder en:

```text
http://localhost:30080
```

Endpoints útiles:

```text
http://localhost:30080/v0/ping
http://localhost:30080/v0/health
http://localhost:30080/docs
```

## Como anadir un nuevo MCP server

En este repo el catalogo inicial del registry vive en dos sitios:

- `data/seed.json`: fuente legible y facil de editar.
- `k8s-local/01-registry-seed-configmap.yaml`: copia embebida que realmente monta Kubernetes en `/data/seed.json`.

Si quieres anadir un nuevo MCP, actualiza **ambos archivos** para que el despliegue local quede consistente.

### Paso 1: preparar la entrada del nuevo server

Cada MCP se define como un objeto `ServerJSON` dentro del array JSON. El esquema minimo suele ser:

```json
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-10-17/server.schema.json",
  "name": "io.github.OWNER/REPO",
  "description": "Una linea descriptiva",
  "repository": {
    "url": "https://github.com/OWNER/REPO",
    "source": "github"
  },
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "nombre-paquete",
      "version": "1.0.0",
      "transport": {
        "type": "stdio"
      }
    }
  ]
}
```

Notas utiles:

- `name` debe ser unico dentro del registry.
- `registryType` puede ser `npm`, `pypi`, `oci` o `nuget`.
- Para MCPs remotos (HTTP/SSE), normalmente se usa `remotes` en vez de `packages`.
- Si el MCP necesita variables de entorno o argumentos de ejecucion, anadelos en `environmentVariables` o `runtimeArguments`.

### Paso 2: editar el seed fuente

1. Abre `data/seed.json`.
2. Anade el nuevo objeto al array.
3. Verifica que el JSON sigue siendo valido.

### Paso 3: sincronizar el ConfigMap de Kubernetes local

1. Abre `k8s-local/01-registry-seed-configmap.yaml`.
2. Copia el contenido actualizado de `data/seed.json` dentro de `data.seed.json` respetando la indentacion YAML.

Importante: editar solo `data/seed.json` **no** cambia lo que monta el pod. Kubernetes usa el JSON embebido en el `ConfigMap`.

### Paso 4: reaplicar los manifiestos

```text
kubectl apply -k k8s-local
```

### Paso 5: forzar la reimportacion del seed

En local, PostgreSQL usa `emptyDir`. El seed solo se importa si la base esta vacia, asi que debes resetear la base reiniciando el pod de Postgres:

```text
kubectl delete pod -n mcp-registry -l app.kubernetes.io/component=database
kubectl rollout status deployment/postgres -n mcp-registry --timeout=180s
kubectl rollout status deployment/registry -n mcp-registry --timeout=180s
```

Al recrearse el pod de Postgres, la base quedara vacia y el registry volvera a aplicar migraciones e importar el nuevo seed.

### Paso 6: validar que el nuevo MCP aparece

Comprueba primero salud del servicio:

```text
http://localhost:30080/v0/health
```

Y despues el listado completo:

```text
http://localhost:30080/v0/servers?limit=100
```

Si quieres validar solo un server concreto:

```text
http://localhost:30080/v0/servers/io.github.microsoft%2Fplaywright-mcp/versions
```

### Alternativa: publicar via API en vez de tocar el seed

Si solo quieres hacer una prueba rapida, puedes publicar un server por API en caliente. Ojo: con auth anonima solo funciona para namespaces `io.modelcontextprotocol.anonymous/*`.

El enfoque con `seed.json` sigue siendo el recomendado cuando quieres que el catalogo quede versionado en Git y se recree igual en cualquier despliegue.

## Notas importantes

- Cambia los valores de `03-secrets.yaml` antes de usarlo más allá de una prueba rápida.
- Como PostgreSQL usa `emptyDir`, si borras el pod perderás la base de datos y el seed se volverá a importar en el siguiente arranque.
- Si el puerto `30080` te molesta, puedes cambiarlo en `07-registry-service.yaml`.
