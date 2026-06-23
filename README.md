# StockAnalyzer — Helm Chart

Chart de Helm para desplegar la imagen
[stockanalyzer-gitops](https://github.com/MAS26-grupo4/stockanalyzer-gitops)
sobre Kubernetes, consumido por Argo CD desde este mismo repositorio.

## Estructura del repositorio

```
.
├── charts/
│   └── stockanalyzer/              # El chart en sí
│       ├── Chart.yaml
│       ├── values.yaml             # defaults
│       └── templates/              # namespace, deployment, service, hpa
├── environments/
│   ├── dev/values.yaml             # Overrides para desarrollo
│   └── prod/values.yaml            # Overrides para producción
├── argocd/
│   ├── application-dev.yaml         # Application de Argo CD para dev
│   └── application-prod.yaml        # Application de Argo CD para prod
└── .github/workflows/
    └── helm.yaml                   # CI/CD: helm lint + template
```

## Ambientes

| Ambiente | Namespace en el cluster | Réplicas | Imagen              | HPA                | Release Helm              |
|----------|--------------------------|----------|---------------------|---------------------|---------------------------|
| dev      | `stockanalyzer-dev`      | 1        | `v1.3.0`           | deshabilitado      | `stockanalyzer`           |
| prod     | `stockanalyzer-prod`     | 3        | `v1.3.0`           | 3-10, CPU 70%       | `stockanalyzer-prod`      |

> Los **namespaces target** están definidos en `argocd/application-dev.yaml`
> y `argocd/application-prod.yaml`. Argo CD los crea automáticamente
> gracias a `syncOptions: CreateNamespace=true` en la Application.
>
> Los **nombres de release de Helm** son distintos para que Helm
> pueda gestionar los dos deployments como releases separados
> (mismo chart, distinto namespace, distinto release name).

## Diferencia con el repo de YAMLs planos

[`stockanalyzer-gitops-manifests`](https://github.com/MAS26-grupo4/stockanalyzer-gitops-manifests)
contiene la primera versión de los manifiestos, escritos a mano
(`kubectl apply -f`). Este repo es la **evolución** de esa primera
versión usando Helm + Argo CD. Los dos repos coexisten: el primero
queda como referencia histórica, este es el que se usa para
desplegar.

## Uso local con Helm (sin Argo CD)

Útil para validar que el chart funciona **antes** de que Argo CD lo
gestione.

```bash
# Validar el chart
helm lint ./charts/stockanalyzer

# Renderizar el YAML que se aplicaría al cluster (default)
helm template stockanalyzer ./charts/stockanalyzer

# Renderizar para dev
helm template stockanalyzer ./charts/stockanalyzer \
  -f charts/stockanalyzer/values.yaml \
  -f environments/dev/values.yaml

# Renderizar para prod
helm template stockanalyzer ./charts/stockanalyzer \
  -f charts/stockanalyzer/values.yaml \
  -f environments/prod/values.yaml

# Desplegar dev a mano (sin Argo CD)
helm upgrade --install stockanalyzer ./charts/stockanalyzer \
  -n stockanalyzer-dev --create-namespace \
  -f charts/stockanalyzer/values.yaml \
  -f environments/dev/values.yaml

# Desplegar prod a mano (sin Argo CD)
helm upgrade --install stockanalyzer-prod ./charts/stockanalyzer \
  -n stockanalyzer-prod --create-namespace \
  -f charts/stockanalyzer/values.yaml \
  -f environments/prod/values.yaml
```

## Uso con Argo CD (recomendado para producción)

Argo CD está configurado para leer este repo y aplicar el chart
automáticamente. **No hace falta correr `helm install` a mano** — las
Applications se encargan de todo, incluyendo:

- Detectar cambios en el repo (polling cada 3 min).
- Renderizar el chart con los values del ambiente.
- Aplicar los recursos al cluster.
- **Self-heal**: revertir cualquier cambio manual al estado del repo.

### Cómo aplicar las Applications por primera vez

```bash
# Argo CD debe estar instalado en el cluster (ver paso 4.2 del guion)
kubectl apply -f argocd/application-dev.yaml
kubectl apply -f argocd/application-prod.yaml
```

Después de aplicadas, las Applications aparecen en la UI de Argo CD
(`http://localhost:8080` con port-forward) y empiezan a sincronizar
en ~3 min. **El primer sync adopta los recursos existentes** (si los
hubiera) sin recrearlos.

## Validar el deploy

```bash
# Estado de las Applications
kubectl -n argocd get application

# Estado del cluster
helm list -A
kubectl -n stockanalyzer-dev get all
kubectl -n stockanalyzer-prod get all

# HPA de prod
kubectl -n stockanalyzer-prod get hpa
```

## Actualizar la versión de la imagen

```bash
# Editar el tag en el values del ambiente correspondiente
$EDITOR environments/dev/values.yaml
$EDITOR environments/prod/values.yaml

# Commit + push
git add environments/
git commit -m "chore: bump to v1.x.x"
git push origin main
```

Argo CD detecta el cambio en su próximo ciclo de polling y aplica el
rolling update. **No hay que correr `helm upgrade` ni `kubectl apply`
manualmente** — eso es justamente GitOps.

## CI/CD

`.github/workflows/helm.yaml` corre en cada push a `main` y en cada
PR. Hace `helm lint` y `helm template` para los tres flavors
(default, dev, prod). Si el template falla, el PR queda bloqueado.

## Recursos

- App: <https://github.com/MAS26-grupo4/stockanalyzer-gitops>
- Manifests planos (referencia histórica): <https://github.com/MAS26-grupo4/stockanalyzer-gitops-manifests>
- Imágenes: <https://github.com/orgs/MAS26-grupo4/packages/container/stockanalyzer-gitops/settings>
- ADRs: <https://github.com/MAS26-grupo4/stockanalyzer-gitops/tree/main/docs/adr>
