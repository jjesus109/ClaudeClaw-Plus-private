# Feature: GHCR Container Registry for ai-hub

**Date:** 2026-09-17  
**Type:** Feature  
**Repo:** ai-hub (github.com/jjesus109/ai-hub)  
**Status:** COMPLETE ✅

---

## Goal

Replace manual `docker build → k3s ctr import` workflow with:
- GitHub Actions builds + pushes image to GHCR on every push to main
- k3s pulls from `ghcr.io/jjesus109/ai-hub:latest` automatically
- No more manual image management on server

---

## SDLC Stages

| Stage | Status |
|-------|--------|
| 1. Inspect | ✅ Done — no CI, no registry, build is manual |
| 2. Plan | ✅ Done — this file |
| 3. Implement | ✅ Done — CI workflow, imagePullSecrets, manifest updated |
| 4. Write tests | ✅ Done — pod pulls from GHCR, healthz 200 |
| 5. Tests pass | ✅ Done |
| 6. Security review | ✅ Done — 2 workflow findings fixed (push-only gates) |
| Push to main | ✅ Done — commits 2401416, cf52b21, 247675b, aa85c5a, b677ade |

---

## Steps

1. Create GitHub Actions workflow — build + push to GHCR [ FINISHED ]

   File: `.github/workflows/docker-publish.yml`
   - Trigger: push to `main`
   - Auth: `GITHUB_TOKEN` (built-in, no secrets needed for GHCR)
   - Build: `docker buildx build --platform linux/amd64`
   - Push: `ghcr.io/jjesus109/ai-hub:latest` + `ghcr.io/jjesus109/ai-hub:<sha>`

2. GHCR package visibility [ FINISHED ] — went PRIVATE (not public). Auth via k8s imagePullSecrets (ghcr-pull-secret in ai-hub namespace) with PAT read:packages scope. registries.yaml approach dropped (deprecated in containerd v2).

   Option A (simpler): set package visibility to public in GitHub → no auth needed on server  
   Option B: create PAT with `read:packages` → add as k3s registry secret

3. Update k3s deployment manifest [ FINISHED ]

   `infra/k3s/ai-hub-deployment.yaml`:
   - `image: ghcr.io/jjesus109/ai-hub:latest`
   - `imagePullPolicy: Always` (pull on every restart to get latest)
   - Add `imagePullSecrets` if going private (Option B)

4. Configure k3s pull auth (private) [ FINISHED ] — used k8s imagePullSecrets (NOT registries.yaml — deprecated in containerd v2). Secret ghcr-pull-secret created in ai-hub namespace. Deployment manifest updated with imagePullSecrets ref. Commit cf52b21.

   ```bash
   # On server — create /etc/rancher/k3s/registries.yaml
   sudo tee /etc/rancher/k3s/registries.yaml <<EOF
   configs:
     ghcr.io:
       auth:
         username: jjesus109
         password: <PAT with read:packages>
   EOF
   sudo systemctl restart k3s
   ```

5. Push workflow + manifest changes to main [ FINISHED ] — commit 2401416, CI triggered

   ```bash
   git add .github/workflows/docker-publish.yml infra/k3s/ai-hub-deployment.yaml
   git commit -m "ci: add GHCR publish workflow + update k3s to pull from registry"
   git push origin main
   ```
   → GitHub Actions triggers, image lands at `ghcr.io/jjesus109/ai-hub:latest`

6. Verify first CI build [ FINISHED ] — CI built successfully, image at ghcr.io/jjesus109/ai-hub:latest + sha-2401416

   Check Actions tab: https://github.com/jjesus109/ai-hub/actions  
   Confirm image appears at: https://github.com/jjesus109/ai-hub/pkgs/container/ai-hub

7. Deploy to k3s from GHCR [ FINISHED ] — pod pulled ghcr.io/jjesus109/ai-hub:latest in 4.7s, Running. Used imagePullSecrets (registries.yaml deprecated in containerd v2). Commit cf52b21.

   ```bash
   ssh javi-server@192.168.100.131
   cd ~/Projects/ai-hub
   git pull origin main
   sudo kubectl apply -f infra/k3s/ai-hub-deployment.yaml -n ai-hub
   sudo kubectl rollout restart deployment/ai-hub -n ai-hub
   sudo kubectl get pods -n ai-hub
   ```

8. Write tests [ FINISHED ] — verified: pod pulled from GHCR (not local cache), healthz 200, no ErrImageNeverPull

9. Tests pass [ FINISHED ]

10. Security review [ FINISHED ] — 2 workflow findings fixed (explicit push-only gates). 1 doc gap added to tech-pendings. Commit 247675b.

11. Push to main [ FINISHED ] — all commits pushed: 2401416, cf52b21, 247675b

---

## Expected Outcome

- Every push to main → CI builds + pushes to GHCR automatically
- k3s pulls fresh image on `kubectl rollout restart` — no manual build/import
- Clean deploy story: `git push` → image ready → `kubectl rollout restart`
