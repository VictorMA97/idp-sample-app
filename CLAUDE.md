# idp-sample-app

App de ejemplo (FastAPI, Python 3.13) que recorre el golden path de `idp-platform`. Repo público.
Idioma de trabajo: español.

## Qué hay

- `app/main.py`: `/`, `/healthz`, `/version`, `/metrics` (Prometheus; la etiqueta `path` usa la ruta declarada, no la URL real).
- `tests/`: pytest con `fastapi.testclient` (requiere `httpx2`, no `httpx`, con la versión de Starlette fijada).
- `Dockerfile`: multi-stage, usuario no-root UID 10001, puerto 8080. Pendiente fijar la imagen base por digest.
- `chart/`: Helm. Deployment endurecido (restricted: no-root, rootfs de solo lectura, sin capabilities, seccomp),
  Service, Ingress (Traefik), NetworkPolicy (permite `kube-system`), ServiceMonitor opcional.
  `values-dev.yaml` / `values-prod.yaml` los **reescribe el CI** (repository, tag y digest): no editar a mano esos campos.
- `.github/workflows/ci.yaml`: test + `helm lint/template` → (PR) scan | (main) build → scan (Trivy) → sign (SBOM + Cosign keyless) → deploy-dev (commit del digest).
- `.github/workflows/promote.yaml`: manual dev→prod; verifica firma y SBOM contra la identidad `ci.yaml@refs/heads/main`.

## Desarrollo

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
pytest -q
```

## Reglas

- Todas las acciones de GitHub fijadas por SHA de commit con el tag como comentario. No sustituir por tags.
- Permisos por job mínimos (`permissions: {}` global). Sin `pull_request_target`.
- Dependencias fijadas en `requirements*.txt`; regenerarlas desde un entorno limpio, no a mano.
- Despliegue siempre por digest (`repo:tag@sha256:...`).

## Estado y pendientes

- Entregado sin ejecutar en GitHub: el pipeline y el chart **no se han validado con `helm` ni en Actions todavía**.
  Primer paso al retomar: push a `main`, mirar el resultado de `ci`, corregir lo que falle.
- Pasos manuales de una vez: paquete GHCR en público, entorno `production` con revisores obligatorios.
- Pendiente: verificar el SHA interno de `setup-trivy` (o cambiar a Grype), imagen base por digest,
  Dependabot/Renovate, `ServiceMonitor` y dashboard (Fase 6), verificación de firma en admisión con Kyverno (Fase 5).
