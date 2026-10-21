# Kubernetes setup

How PatientRepo is built, shipped and run on the home k3s cluster, the decisions behind it, and what is left to do.

_Last updated: 2026-10-21_

## Status at a glance

| Item | Status |
|---|---|
| k3s cluster on `max-ga-server` | ✅ Running |
| `kubectl` / k9s / Headlamp access from the dev laptop | ✅ Working |
| API image built and pushed to GHCR (`build-api` workflow) | ✅ Working |
| Front-end image built and pushed to GHCR (`build-frontend` workflow) | ✅ Working (since `fa02bbc`) |
| GHCR pull credentials on the node (`registries.yaml`) | ⚠️ File present, not yet proven by a real pull |
| Kubernetes files in `deploy/k8s/` | ✅ Written, pass `kubectl apply --dry-run=client` |
| First deploy (`kubectl apply -k deploy/k8s`) | ⏳ Not yet applied |
| ArgoCD (GitOps) | ❌ Not installed |
| Automatic image-tag bump from CI | ❌ Not started |
| Prometheus / Grafana | ❌ Not started |

## Cluster

| | |
|---|---|
| Host | FX Desktop, hostname `max-ga-server` |
| IP | `192.168.20.18` (DHCP; see TODOs: reserve it) |
| OS | Ubuntu 26.04 LTS, kernel 7.0 |
| Distribution | k3s `v1.36.3+k3s1`, single node (control plane and worker) |
| Container runtime | containerd 2.3 (bundled with k3s) |
| Ingress | Traefik 3.7 (bundled with k3s), `LoadBalancer` on `192.168.20.18:80/443` |
| Metrics | metrics-server (bundled). `kubectl top` and resource views in k9s and Headlamp work |
| Storage | `local-path` provisioner (bundled) |
| Namespaces | Defaults, plus `databases` (empty, reserved) and `patientrepo` (this app) |

Host install steps are in the separate `homelab` repo (`KubernetesInstall.md`).

### Access from the dev laptop

The kubeconfig was copied from the server's `/etc/rancher/k3s/k3s.yaml` to `~/.kube/config`, with `127.0.0.1` replaced by `192.168.20.18`. The k3s API certificate already includes `192.168.20.18` and `max-ga-server`, so no `--tls-san` is needed for those.

To look at the cluster:

- `kubectl`: the command line.
- `k9s`: a terminal UI, like lazydocker. Type `:pods`, press `0` for all namespaces, `l` for logs, `s` for a shell.
- Headlamp: a desktop GUI. It reads the same kubeconfig.

## Build and delivery pipeline

```
git push (main)
  └─ GitHub Actions (GitHub-hosted ubuntu-latest runners)
       ├─ .github/workflows/api.yml       paths: src/backend/**   → ghcr.io/maxtsimpson/patientrepo-api
       └─ .github/workflows/frontend.yml  paths: src/frontend/**  → ghcr.io/maxtsimpson/patientrepo-fe
            tags: sha-<7-char commit>, latest
  └─ k3s pulls from GHCR using node-level credentials
  └─ (planned) ArgoCD syncs deploy/k8s from this repo
```

- Pushing to GHCR uses the workflow's built-in `GITHUB_TOKEN` (`permissions: packages: write`). No personal token is stored in GitHub.
- The packages must stay **Private**. The repo is public, and a package's visibility is set separately from its repo's.

To build locally:

```bash
# API: run from the repo root. The build context is src/backend
docker build -f src/backend/PatientRepo.Api/Dockerfile -t patientrepo-api src/backend

# Front end: run from src/frontend
docker build -t patientrepo-fe src/frontend
```

### Registry credentials on the node

`/etc/rancher/k3s/registries.yaml` on the server (root-only, mode 600):

```yaml
configs:
  "ghcr.io":
    auth:
      username: maxtsimpson
      password: <classic PAT with read:packages only>
```

Run `sudo systemctl restart k3s` after editing it. To test a pull on the server, run `sudo k3s crictl pull ghcr.io/maxtsimpson/patientrepo-api:<tag>`.

## Kubernetes files (`deploy/k8s/`)

```
deploy/k8s/
├── kustomization.yaml     # namespace, common labels, image tags
├── namespace.yaml         # patientrepo
├── ingressroute.yaml      # Traefik IngressRoute and the strip-/backend Middleware
├── backend/
│   ├── deployment.yaml    # patientrepo-api: port 8080, uid 1654
│   └── service.yaml       # port 80 → http (8080)
└── frontend/
    ├── deployment.yaml    # patientrepo-fe: port 3000, uid 1000
    └── service.yaml       # port 80 → http (3000)
```

### Routing (`http://192.168.20.18`)

| Path | Goes to | Notes |
|---|---|---|
| `/backend/*` | `patientrepo-api` (.NET) | `/backend` is stripped, so `/backend/foo` reaches the API as `/foo` |
| `/*` | `patientrepo-fe` (Bun) | Includes the Bun server's own `/api/*` routes |

### Deploy and check by hand

```bash
kubectl apply -k deploy/k8s
kubectl -n patientrepo get pods -w
curl http://192.168.20.18/            # front end
curl http://192.168.20.18/api/hello   # Bun server route
curl http://192.168.20.18/backend/    # .NET API → "Hello World!"
```

To change an image tag: edit `images:` in `kustomization.yaml`, or run `cd deploy/k8s && kustomize edit set image ghcr.io/maxtsimpson/patientrepo-api=ghcr.io/maxtsimpson/patientrepo-api:sha-<commit>`.

### Troubleshooting

| Symptom | Likely cause / where to look |
|---|---|
| `ImagePullBackOff` / `ErrImagePull` | The tag doesn't exist in GHCR, the PAT has expired, or `registries.yaml` is wrong. Run `kubectl -n patientrepo describe pod <pod>` |
| `CreateContainerConfigError: ... runAsNonRoot` | The image runs as root, or as a user name rather than a numeric ID. Set `runAsUser` (see the decisions below) |
| Front-end pod slow to become Ready | Bun bundles the React and Tailwind code on startup. The startupProbe allows up to 120 s |
| 404 from Traefik | Check the IngressRoute exists: `kubectl -n patientrepo get ingressroute`. Check the Traefik logs: `kubectl -n kube-system logs deploy/traefik` |

## Decisions

These are recorded so they aren't re-argued or silently undone. Change them on purpose and update this section when you do.

| # | Decision | Why | Revisit when |
|---|---|---|---|
| D1 | **Single-node k3s** on the FX Desktop | Lightweight, and comes with Traefik, metrics-server and local-path storage | A second machine joins |
| D2 | **GitHub-hosted runners** for CI, not self-hosted (yet) | Fewer moving parts. Free for a public repo | Builds need LAN access, or a self-hosted runner is set up (see the TODOs) |
| D3 | **Image tags `sha-<commit>`** are pinned in Kustomize. `latest` is also pushed but never deployed | Reproducible deploys. With `latest`, git never changes, so ArgoCD would never redeploy | No change planned |
| D4 | **Node-level GHCR credentials** (`registries.yaml`) instead of per-namespace `imagePullSecrets` | Single node. Every namespace (and ArgoCD apps later) can pull without copying secrets around | The cluster has several nodes, or needs credentials per tenant |
| D5 | **Path-based routing on the node IP** (`/backend` → API, `/` → front end) | Needs no DNS. One address for the browser, so no CORS | Local DNS is set up (see the TODO on hostnames and TLS) |
| D6 | API mounted at **`/backend`**, not `/api` | The Bun front-end server owns `/api/*` for its own server-side routes | The front end stops serving its own API |
| D7 | **Traefik `IngressRoute` and `Middleware` types** instead of a standard `Ingress` | Each route can have its own middleware (strip-prefix only on `/backend`) | Moving away from Traefik |
| D8 | **Kustomize** (no Helm chart) for the app's files | Plain YAML, built into `kubectl`, and CI can set tags with `kustomize edit set image` | Configuration needs to differ a lot between environments |
| D9 | **Deploy files live in this repo** (`deploy/k8s/`) | One repo while it's a solo project | CI tag-bump commits get noisy. Then move them to a GitOps repo (e.g. `homelab`) |
| D10 | **Front end runs as a Bun server** (`bun src/index.ts`), not static files on nginx | Needed for server-side routes and components | Image size (~670 MB) becomes a problem. See the TODO on slimming the image |
| D11 | **API runs as uid 1654** through the pod `securityContext` | The aspnet image creates `app` (1654) but defaults to root. `runAsNonRoot` needs a numeric non-root user | `USER $APP_UID` is added to the Dockerfile (the pod setting can then stay as a safeguard) |
| D12 | **Front end runs as uid 1000** through the pod `securityContext` | The image's `USER bun` is a name, which `runAsNonRoot` can't check | No change planned |
| D13 | **Container ports: API 8080, front end 3000.** Services expose port 80 | 8080 is the .NET 8+ default for non-root containers. 3000 is Bun's default | No change planned |
| D14 | **API Docker build context is `src/backend`**, with the Dockerfile next to the project | The API needs sibling projects (Models etc.), and the front end's files stay out of the context | No change planned |
| D15 | **Secrets never go in git** (`appsettings*.json` holds only non-secret settings) | Public repo, and the domain is patient data | No change planned |

## TODOs

### Next up
- [ ] Run the first deploy: `kubectl apply -k deploy/k8s`, then check all three URLs. This also proves the GHCR pull credentials work.
- [ ] Commit `deploy/k8s/` and this doc.
- [ ] Install **ArgoCD** (Helm or the official install file) and create an Application for `maxtsimpson/PatientRepo` → `deploy/k8s`, with automatic sync and self-heal turned on.
- [ ] **CI tag bump:** after pushing an image, each workflow runs `kustomize edit set image ...:sha-<commit>` and commits the change, so ArgoCD redeploys. Guard against CI loops: the commit touches only `deploy/**`, which neither workflow's `paths:` filter matches. The alternative is Argo CD Image Updater.

### Image and Dockerfile hygiene
- [ ] Add `USER $APP_UID` to the runtime stage of `src/backend/PatientRepo.Api/Dockerfile` (see D11).
- [ ] Fix the `LABEL` syntax in both Dockerfiles: `LABEL key="value"`, not the old `LABEL key "value"` form.
- [ ] Add a label for the base image (`org.opencontainers.image.base.name`) so tools can show what each image was built from.
- [ ] Slim the front-end image (~670 MB), e.g. `bun build --compile` to a single executable on a smaller base image.

### App readiness
- [ ] Give the API real health endpoints (`/healthz` for liveness, `/readyz` for readiness) instead of probing `/`.
- [ ] Tune CPU and memory requests and limits from real usage (`kubectl top pods -n patientrepo`).
- [ ] Choose how secrets get into the cluster: Sealed Secrets, or SOPS with age, so encrypted secrets can live in git for ArgoCD.
- [ ] Database: decide what runs in the `databases` namespace, and how it's stored (local-path vs NFS) and backed up.

### Cluster and infrastructure
- [ ] Reserve `192.168.20.18` for the server in the router's DHCP settings (router `192.168.20.1`). The kubeconfig, SSH and routing all depend on it.
- [ ] The GHCR read token expires. Note the date, set a reminder, and put the renewal steps here.
- [ ] Hostnames and TLS: local DNS (e.g. `patientrepo.home.lan`) plus cert-manager or Traefik's built-in certificates. Then move routing from paths to hosts (D5).
- [ ] **Monitoring:** the `kube-prometheus-stack` Helm chart (Prometheus and Grafana).
- [ ] **NFS:** the NFS server is on the host. Add an NFS storage provisioner if pods need shared or persistent storage beyond local-path.
- [ ] Pin and plan k3s upgrades (`INSTALL_K3S_VERSION`, or the system-upgrade-controller).
- [ ] Back up the cluster state (k3s stores it in SQLite on a single node), plus volume backups.
- [ ] Self-hosted runner: `~/actions-runner` on the server is registered to `max-enterprises/TheatreManagement`, shows as offline, and isn't a service. For PatientRepo, either register a runner for this repo, or move the repo into the `max-enterprises` org and use an org-level runner. Then install it as a service (`sudo ./svc.sh install`).
