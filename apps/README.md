# Client Apps

This directory contains client-specific applications that deploy alongside the base chart.

## Structure

```
apps/
├── _app-template/          # Template for creating new apps
│   ├── helmrelease.yaml
│   ├── namespace.yaml
│   ├── gitrepository.yaml
│   └── kustomization.yaml
├── microservices-demo/     # Demo app (Google Online Boutique)
│   ├── namespace.yaml
│   ├── gitrepository.yaml
│   ├── helmrelease.yaml
│   ├── certificate.yaml
│   ├── ingressroute.yaml
│   ├── http-redirect.yaml
│   └── kustomization.yaml
└── <your-app>/             # Your custom apps go here
```

## Included Apps

### microservices-demo (Online Boutique)
Google Cloud Platform's sample microservices application consisting of 11 services.
- **Source**: https://github.com/GoogleCloudPlatform/microservices-demo
- **Version**: v0.10.4
- **Ingress**: `boutique.<domain>` (patched per environment in overlays)

## Adding a New App

1. Copy `_app-template/` to `<your-app-name>/`
2. Modify the YAML files with your app configuration
3. Add the app directory to `kustomization.yaml` resources
4. Create environment-specific patches in overlays if needed

## Environment-Specific Configuration

Domain and other environment-specific settings are patched in the overlays:
- `overlays/dev/kustomization.yaml` - patches for dev environment
- `overlays/prod/kustomization.yaml` - patches for prod environment
