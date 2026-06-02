# MCP Registry en Kubernetes (EKS)

Este directorio contiene los manifiestos para desplegar `mcp-registry` y `postgres` en Kubernetes, separados por tipo de recurso.

## Archivos

- `00-namespace.yaml`: namespace dedicado `mcp-registry`.
- `01-registry-seed-configmap.yaml`: incluye el `seed.json` y lo monta como volumen de solo lectura en `/data/seed.json`.
- `02-registry-configmap.yaml`: variables no sensibles del registry.
- `03-secrets.yaml`: credenciales de Postgres y secretos del registry.
- `04-postgres-statefulset.yaml`: base de datos PostgreSQL con almacenamiento persistente `gp3`.
- `05-postgres-service.yaml`: servicio interno de PostgreSQL.
- `06-registry-deployment.yaml`: despliegue del MCP Registry con `initContainer` que espera a PostgreSQL.
- `07-registry-service.yaml`: servicio interno del registry.
- `08-ingress.yaml`: ingress para EKS usando AWS Load Balancer Controller (`ingressClassName: alb`).
- `09-kustomization.yaml`: manifiesto Kustomize numerado, alineado con el resto de archivos.
- `kustomization.yaml`: punto de entrada estándar para `kubectl apply -k k8s`.

## Catalogo actual

Servers sembrados en `01-registry-seed-configmap.yaml`:

- `io.github.github/github-mcp-server` v0.19.1 — local, paquete OCI `ghcr.io/github/github-mcp-server` (transport `stdio`, requiere `GITHUB_PERSONAL_ACCESS_TOKEN`).
- `io.github.github/github-mcp-server-remote` v0.19.1 — remoto hosted en `https://api.githubcopilot.com/mcp/` (transport `streamable-http`, sin Docker local).
- `io.github.microsoft/playwright-mcp` v0.0.75 — npm `@playwright/mcp` (transport `stdio`).

## Requisitos en EKS

Antes de aplicar estos manifiestos, asegúrate de tener:

1. Un clúster EKS operativo.
2. El AWS Load Balancer Controller instalado si vas a usar el `Ingress` ALB.
3. Una `StorageClass` llamada `gp3` disponible.
4. Los valores reales ajustados en `03-secrets.yaml`.
5. Si usas HTTPS, un certificado ACM y las anotaciones correspondientes en `08-ingress.yaml`.

## Despliegue

Puedes aplicar todo con Kustomize:

```text
kubectl apply -k k8s
```

O aplicar archivo por archivo:

```text
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-registry-seed-configmap.yaml
kubectl apply -f k8s/02-registry-configmap.yaml
kubectl apply -f k8s/03-secrets.yaml
kubectl apply -f k8s/05-postgres-service.yaml
kubectl apply -f k8s/04-postgres-statefulset.yaml
kubectl apply -f k8s/07-registry-service.yaml
kubectl apply -f k8s/06-registry-deployment.yaml
kubectl apply -f k8s/08-ingress.yaml
```

## Como anadir un nuevo MCP server

En este repo el catalogo inicial del registry vive en dos sitios:

- `data/seed.json`: fuente legible y facil de mantener.
- `k8s/01-registry-seed-configmap.yaml`: copia embebida que Kubernetes monta en `/data/seed.json`.

Si necesitas anadir un nuevo MCP al despliegue de EKS, actualiza **ambos archivos** para evitar que el repositorio quede desincronizado.

### Paso 1: preparar la definicion del server

Cada entrada del seed es un objeto `ServerJSON`. Un ejemplo minimo:

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

Ten en cuenta lo siguiente:

- `name` debe ser unico.
- `registryType` puede ser `npm`, `pypi`, `oci` o `nuget`.
- Si el MCP es remoto (HTTP/SSE), normalmente se modela con `remotes` y no con `packages`.
- Si necesita secretos, variables o argumentos de ejecucion, incluyelos en `environmentVariables` y `runtimeArguments`.

### Paso 2: editar el seed fuente

1. Edita `data/seed.json`.
2. Anade el nuevo server al array.
3. Valida que el JSON siga siendo correcto.

### Paso 3: sincronizar el ConfigMap de EKS

1. Edita `k8s/01-registry-seed-configmap.yaml`.
2. Sustituye el contenido de `data.seed.json` por la version actualizada del seed, respetando la indentacion YAML.

Importante: aplicar solo `data/seed.json` no cambia nada en el cluster. El pod lee el seed desde el `ConfigMap` ya renderizado en `k8s/01-registry-seed-configmap.yaml`.

### Paso 4: aplicar el cambio al cluster

```text
kubectl apply -k k8s
```

### Paso 5: decidir como quieres cargar el nuevo seed

Aqui hay dos escenarios:

#### Opcion A: recrear la base y reimportar todo el seed

Es la opcion mas simple cuando estas en un entorno de prueba o cuando puedes permitirte reconstruir el catalogo completo.

1. Escala o elimina temporalmente el registry.
2. Elimina el `PersistentVolumeClaim`/volumen de PostgreSQL o recrea el `StatefulSet` con almacenamiento vacio.
3. Levanta de nuevo PostgreSQL y el registry.
4. El registry aplicara migraciones e importara el seed actualizado sobre una base vacia.

Este enfoque es equivalente a "resetear la base" y debe hacerse con cuidado en EKS, porque implica perder el catalogo existente.

#### Opcion B: mantener la base y publicar en caliente por API

Si no quieres vaciar la base, no basta con aplicar el nuevo `ConfigMap`: el registry **no reimporta** automaticamente el seed si la BD ya contiene datos.

En ese caso tienes dos alternativas:

- publicar el nuevo server por API (`POST /v0/publish`), o
- preparar una operacion controlada para vaciar y reconstruir el catalogo.

Para namespaces `io.github.*/*` necesitaras OAuth GitHub real; la autenticacion anonima solo sirve para `io.modelcontextprotocol.anonymous/*`.

### Paso 6: validar que el nuevo MCP aparece

Comprueba salud:

```text
GET /v0/health
```

Lista todos los servers:

```text
GET /v0/servers?limit=100
```

Y valida el detalle de uno concreto:

```text
GET /v0/servers/{name}/versions
```

En local las URLs completas serian, por ejemplo:

```text
http://localhost:30080/v0/servers?limit=100
http://localhost:30080/v0/servers/io.github.microsoft%2Fplaywright-mcp/versions
```

En EKS usa tu dominio real o el host configurado en el `Ingress`.

## Consideraciones importantes

- El seed solo se carga cuando la base de datos está vacía. Si cambias `seed.json`, tendrás que recrear el volumen de PostgreSQL para forzar la reimportación.
- La versión del registry fijada y verificada en estos manifiestos es `ghcr.io/modelcontextprotocol/registry:1.7.8`.
- He dejado `MCP_REGISTRY_OIDC_ENABLED=false` por defecto, alineado con una instalación inicial más simple. Si necesitas OIDC, completa las variables en `02-registry-configmap.yaml` y `03-secrets.yaml`.
- El `Ingress` usa el host de ejemplo `mcp-registry.example.com`; cámbialo por tu dominio real.
- Si prefieres exponer el servicio mediante un `Service` tipo `LoadBalancer` en lugar de `Ingress`, se puede añadir fácilmente, pero aquí he priorizado el patrón típico de EKS con ALB.
