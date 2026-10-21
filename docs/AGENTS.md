# AGENTS.md

Start here if you're an AI agent (or a new contributor) working in this repo. It points to the docs that hold the standards, setup details and decisions. Read the doc relevant to your task before you change anything in its area.

## Project in one line

PatientRepo is an API for taking in patient and visit data (HL7 / FHIR) from different hospitals. It keeps a central patient repository and a per-hospital location repository. It has a .NET back end and a Bun/React front end in one monorepo, and is deployed to a home k3s cluster. Requirements are in the root [`README.md`](../README.md).

## Doc index

| Doc | Read it when you're… |
|---|---|
| [`README.md`](../README.md) | Working on any feature: business and technical requirements, proposed tech stack |
| [`docs/kubernetes.md`](kubernetes.md) | Touching `deploy/`, Dockerfiles, `.github/workflows/`, image names or tags, ports, routing, or anything cluster-related. It has the **decision log (D1–D15)** and the **TODO list** |
| [`src/frontend/CLAUDE.md`](../src/frontend/CLAUDE.md) | Working in `src/frontend/`: Bun-first conventions (use Bun, not Node / Vite / Express) |
| [`src/frontend/README.md`](../src/frontend/README.md) | Running the front end locally |

When you add a new doc under `docs/`, add a row here.

## Repo map

```
.github/workflows/   api.yml (→ patientrepo-api), frontend.yml (→ patientrepo-fe)
deploy/k8s/          Kustomize files for the k3s cluster (see docs/kubernetes.md)
docs/                Project docs (this file, kubernetes.md)
src/backend/         .NET 10 solution: PatientRepo.slnx, one folder per project
  PatientRepo.Api/   ASP.NET Core minimal API and its Dockerfile
src/frontend/        Bun + React + Tailwind + shadcn/ui, run as a Bun server
```

## Standards

### Back end (.NET)
- .NET 10, and the solution file uses the `.slnx` format (`src/backend/PatientRepo.slnx`).
- **One `.csproj` per folder**, with the folder named after the project (e.g. `src/backend/PatientRepo.Models/PatientRepo.Models.csproj`). Never put two projects in one folder.
- Shared build settings and package versions go in `src/backend/` (`Directory.Build.props` and `Directory.Packages.props` when added), not the repo root.
- When you add a project that the API references, also add its `.csproj` to the restore section of `src/backend/PatientRepo.Api/Dockerfile`.

### Front end
- Follow [`src/frontend/CLAUDE.md`](../src/frontend/CLAUDE.md): Bun for the runtime, tests, bundling and package installs.
- It runs as a Bun server (`bun src/index.ts`), not a static build. Server-side routes live in `src/index.ts` under `/api/*`.

### Containers and deployment
- Only deployable apps get a Dockerfile, and it sits next to the app. The API's build context is `src/backend`. The front end's is `src/frontend`.
- Images: `ghcr.io/maxtsimpson/patientrepo-api` and `ghcr.io/maxtsimpson/patientrepo-fe`, tagged `sha-<7-char commit>`. Deploy files pin `sha-*` tags in `deploy/k8s/kustomization.yaml`. **Never deploy `latest`.**
- Container ports: API **8080**, front end **3000**. Both run as non-root (uids 1654 and 1000).
- Change the cluster by editing `deploy/k8s/`, not with `kubectl edit` or hand-written `kubectl patch` on live objects. Once ArgoCD is installed, it will undo live changes.
- Routing: `/backend/*` goes to the API (prefix stripped), `/*` to the front end. Don't add API routes under `/api` on the .NET side. That path belongs to the Bun server (decision D6).

### Secrets and data
- **This repo is public.** Never commit secrets, connection strings with credentials, tokens, or real patient data, including in sample or test data and in old commits.
- `appsettings*.json` holds only non-secret settings. Locally, use `dotnet user-secrets`. In Docker, use a `.env` file (gitignored). In k3s, use Kubernetes Secrets.

## Working rules for agents
- If you change something covered by `docs/kubernetes.md` (files in `deploy/`, workflows, Dockerfiles, ports, image names, cluster add-ons), update that doc in the same change. That covers the status table, a new or changed decision row, and ticking or adding TODOs.
- Don't reverse a recorded decision on your own. Ask, or propose the change with its reasoning, and record the new decision.
- Check a change before calling it done. Run `docker build` for Dockerfile changes, and `kubectl apply -k deploy/k8s --dry-run=client` for file changes.
- Applying to the cluster, pushing images and committing are the owner's call unless they've asked you to do them.
