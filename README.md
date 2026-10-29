# idp-sample-app

App de ejemplo (FastAPI) que recorre el **golden path** de la plataforma
[idp-platform](https://github.com/VictorMA97/idp-platform):

```
push a main → tests + lint del chart → build → Trivy → publicar en GHCR
            → SBOM → firma Cosign (keyless) → commit del digest en values-dev.yaml
            → Argo CD despliega en dev
manual:  promote-to-prod → verifica firma y SBOM → commit en values-prod.yaml → Argo CD despliega en prod
```

Los servicios que crea Backstage con su plantilla usan esta misma estructura y pipeline.

## Contenido

| Ruta | Qué es |
|---|---|
| `app/` | API con `/`, `/healthz`, `/version` y `/metrics` (Prometheus) |
| `tests/` | Tests con pytest |
| `Dockerfile` | Multi-stage, usuario no-root (UID 10001), sin herramientas de build en la imagen final |
| `chart/` | Chart de Helm: Deployment endurecido, Service, Ingress, NetworkPolicy, ServiceMonitor opcional |
| `.github/workflows/ci.yaml` | Pipeline de CI/CD |
| `.github/workflows/promote.yaml` | Promoción manual dev → prod con verificación de firma |

## Puesta en marcha

Es la Etapa 3 del README de `idp-platform`; hazla después de tener los clústeres y GitOps activos.

1. Crea un repo **público** `idp-sample-app` en GitHub y sube este contenido (rama `main` o `master`).
2. Espera a que `ci` termine en verde. Publica la primera imagen en GHCR.
3. En GitHub: tu perfil → *Packages* → `idp-sample-app` → *Package settings* → **Change visibility → Public**.
   Si no, los clústeres no podrán descargar la imagen (`ImagePullBackOff`). Es un paso manual de una sola vez.
4. En el repo: *Settings → Environments → New environment* → `production` → activa **Required reviewers**.
5. En `idp-platform`: `make sample-app APP_REPO_URL=https://github.com/<usuario>/idp-sample-app.git`, commit y push.
6. Prueba:
   - dev: <http://sample-app.dev.localtest.me:8081>
   - prod (tras ejecutar *Actions → promote-to-prod*): <http://sample-app.prod.localtest.me:8082>

Mientras no exista la primera imagen, los pods quedan en `ImagePullBackOff`: es lo esperado.

## Desarrollo local

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
pytest -q
uvicorn app.main:app --port 8080 --reload
```

## Seguridad del pipeline

- **Acciones fijadas por SHA de commit**, con el tag como comentario. En marzo de 2026 se manipularon los tags de `aquasecurity/trivy-action` (CVE-2026-33634, [GHSA-69fq-xp46-6x23](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23)): un tag se puede mover; un SHA no.
- **Trivy**: acción `v0.35.0` y binario `v0.69.3`, las versiones citadas como limpias tras el incidente. Súbelas de forma deliberada, no automática.
- **Aislamiento de Trivy**: el flujo en la rama principal se divide en `build` (escribe en GHCR) → `scan` (solo lectura del registro) → `sign` (firma). Trivy corre sin `id-token`, sin escritura y sin acceso a la identidad de firma.
- **Permisos mínimos**: `permissions: {}` a nivel de workflow y permisos por job. `id-token: write` solo en el job que firma; `contents: write` solo en el que hace el commit GitOps.
- **Sin `pull_request_target`** (vector del robo de token original en el incidente de Trivy).
- **Puerta de calidad**: si Trivy encuentra CRITICAL/HIGH con parche disponible, no se firma ni se despliega.
- **Firma keyless con Cosign** (identidad OIDC del workflow) y **SBOM SPDX atestado** sobre el digest.
- **Despliegue por digest**: el chart referencia `repo:tag@sha256:...`, inmutable.
- **Admisión en el clúster**: Kyverno (en dev y prod) rechaza, en modo Deny, imágenes de esta app que no estén firmadas por `ci.yaml` de este repo.
- **Promoción verificada**: `promote.yaml` comprueba firma y SBOM contra la identidad exacta de `ci.yaml` en la rama principal antes de tocar prod.

## Pendiente / mejoras

- La plantilla de servicios de `idp-backstage` ya fija `aquasecurity/setup-trivy` por SHA y usa
  `skip-setup-trivy: true`. Si el CI de esta app instala Trivy a través de una acción sin fijar, conviene
  actualizarlo igual.
- Fijar la imagen base del `Dockerfile` por digest.
- Dependabot/Renovate para actualizar acciones y dependencias con revisión.
- `ServiceMonitor` y dashboard de Grafana desde la plantilla (cuando exista la fase de observabilidad).
