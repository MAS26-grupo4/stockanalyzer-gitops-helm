# StockAnalyzer — Helm Chart

Chart de Helm para desplegar la imagen
[stockanalyzer-gitops](https://github.com/MAS26-grupo4/stockanalyzer-gitops)
sobre Kubernetes.

## Estructura

```
.
├── charts/
│   └── stockanalyzer/         # El chart en sí
│       ├── Chart.yaml
│       ├── values.yaml        # defaults
│       └── templates/         # namespace, deployment, service, hpa
├── environments/
│   ├── dev/values.yaml        # Overrides para desarrollo
│   └── prod/values.yaml       # Overrides para producción
└── .github/workflows/
    └── helm.yaml              # CI/CD: helm lint + template
```

## Diferencia con el repo de YAMLs planos

`stockanalyzer-gitops-manifests` contiene la primera versión de los
manifiestos, escritos a mano. Este repo contiene la **evolución** de
esa primera versión usando Helm. Los dos repos coexisten: el primero
queda como referencia histórica, este es el que se usa para desplegar.

## Uso local

```bash
# Validar el chart
helm lint ./charts/stockanalyzer

# Renderizar el YAML que se aplicaría al clúster (default)
helm template stockanalyzer ./charts/stockanalyzer

# Renderizar para dev
helm template stockanalyzer ./charts/stockanalyzer \
  -f charts/stockanalyzer/values.yaml \
  -f environments/dev/values.yaml

# Desplegar a un clúster (ej. minikube)
helm upgrade --install stockanalyzer ./charts/stockanalyzer \
  -n stockanalyzer --create-namespace \
  -f charts/stockanalyzer/values.yaml \
  -f environments/dev/values.yaml
```

## Ambientes

| Ambiente | Réplicas | Imagen | HPA |
|---|---|---|---|
| `default` (values.yaml) | 2 | v1.1.0 | 2-5, CPU 70% |
| `dev` | 1 | dev-latest (pullPolicy: Always) | deshabilitado |
| `prod` | 3 | v1.3.0 (pullPolicy: IfNotPresent) | 3-10, CPU 70% |

## CI/CD

`.github/workflows/helm.yaml` corre en cada push a `main` y en cada PR.
Hace `helm lint` y `helm template` para los tres flavors (default, dev,
prod). Si el template falla, el PR queda bloqueado.
