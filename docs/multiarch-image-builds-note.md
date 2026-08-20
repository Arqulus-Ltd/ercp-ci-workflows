# Change note — multi-architecture (amd64 + arm64) container image builds

**For:** DevOps / platform-devops. **Date:** 2026-08-20. **Trigger:** demo identity DB migration failed at container init with `exec: "/atlas": stat /atlas: no such file or directory`.

This note explains **what is changing and why**, and is self-contained — everything DevOps needs is here or in ECR. It is not a line-by-line diff. The migrate image is already fixed and published; the service-image change is described here for platform-devops to implement in the reusable CI workflow in this repo.

---

## 1. What went wrong

The demo identity migration Job crashed before running a single migration. The error was read as "the image is missing the atlas fix," but the atlas binary was never the problem — it was the **CPU architecture** of the image.

Every image in the estate is built with a plain `docker build` on a single-architecture runner, so each image is **single-arch**. When such an image is scheduled onto a node of a **different** architecture, the container cannot start: the kernel can't load a foreign-architecture binary, which surfaces (confusingly) as a "no such file or directory" on the entrypoint rather than an obvious "wrong architecture" message.

The demo fleet is **mixed / not guaranteed amd64** (arm64 nodepools were recently introduced). So single-arch images are no longer safe — an image built for one arch will fail on the other. Migrate was simply the first casualty because it runs as a PreSync Job, ahead of the services.

## 2. What is changing

Images move from **single-architecture** to **multi-architecture**. Instead of publishing one image for one CPU, the build publishes a **manifest list** — a single tag/digest that contains both an amd64 and an arm64 variant. Each Kubernetes node then automatically pulls the variant matching its own architecture. Nothing in the deployment manifests changes: the Job/Deployment still references one digest; the registry serves the right layers per node.

Two build paths are affected:

| Build path | Owner | Status |
|---|---|---|
| **Migrate image** (built from the app repo) | App/Architecture | **Done.** Multi-arch image published to ECR — digest `sha256:66c1975ba062bcf81fe4b806a2f8eb4e38bddc635a6ec9eb3f9bb6fe6feff4f0` (tag `migrate-53a04d4c`). Repoint the demo migrate Job(s) to this digest and re-run. |
| **Service images** (all 15 services + identity, built via the reusable workflow **in this repo**) | platform-devops | **To do** — same change, described in §4. |

The migrate build lives in the app repo (not accessible to DevOps), but nothing about it is needed here beyond the **digest above**, which is already in the shared ECR registry. Merging the migrate build change is App/Architecture's task; it does not block DevOps.

## 3. Why it is done this way

- **Manifest list, not two tags.** One digest that resolves per-node keeps the GitOps values files, ArgoCD, and rollback flow exactly as they are. There is no per-arch branching in the deployment layer.
- **Emulated cross-build on the existing runner.** The build runner stays amd64; the arm64 variant is produced through emulation during the build. No new arm64 build infrastructure is required just to produce images. (This is separate from the arm64 *runtime* nodepool and the arm64 *ARC runner* work — those are about where workloads and CI jobs run, not about image contents.)
- **Fail-loud build assertion.** The migrate image now runs a tiny check **inside the build, once per architecture**, that the entrypoint binary actually exists and executes on that arch. If a base image ever ships without the expected binary on one architecture, the **build** fails immediately — instead of a broken image reaching a cluster and failing at runtime, which is what happened here. The same guard is worth adding to any image whose entrypoint is a third-party binary.
- **Scan gate preserved.** The vulnerability scan still runs and still blocks publication on HIGH/CRITICAL. It scans the amd64 variant (the two variants carry the same file set, so the vulnerability surface is equivalent), and it runs **before** the multi-arch push, so a failing scan still prevents anything reaching the registry.
- **Digest resolution is unchanged.** The pushed digest is read back from the registry by tag, which returns the manifest-list digest — exactly the value that gets pinned into GitOps. No change to how digests are captured or written back.

## 4. Scope of the service-image change (platform-devops)

The service build logic lives in **one** reusable workflow in this repo, which both the 15-service caller and the identity caller invoke. So this is **a single change that fixes every service at once**, not fifteen changes.

Conceptually, the reusable workflow needs to:
1. **Enable emulated multi-arch building** on the runner (add the standard QEMU + Buildx setup at the top of the job).
2. **Build both architectures and publish them as one manifest list** in the push step, rather than building and pushing a single architecture.
3. **Keep the scan as a gate before publication** — build the amd64 variant locally first, scan it, and only then produce and push the multi-arch manifest.
4. **Leave the digest capture and GitOps write-back untouched** — they already resolve the manifest-list digest by tag.

### Reference sequence (as implemented for the migrate image)

This is the exact flow already proven on the migrate image, described so it can be reproduced here without needing the app repo:

1. At the top of the build job, add the two standard setup steps — **QEMU** (for cross-arch emulation) and **Buildx** (the extended builder).
2. **Build the amd64 variant locally** (loaded into the runner's Docker) and run the **vulnerability scan** against it. This keeps the scan as a gate before anything is pushed.
3. **Build both architectures together and push them as a single manifest list** to ECR in one step. The builder produces both variants (the arm64 one under emulation) and pushes the combined manifest; the old separate `docker push` step is no longer needed.
4. **Read the pushed digest back from the registry by tag** exactly as today — it now returns the manifest-list digest — and write it into GitOps unchanged.

Additionally, if a service's entrypoint is a third-party binary, add a one-line build-time step that executes that binary during the build; under the multi-arch build it runs once per architecture, so a base missing the binary on either arch fails the build instead of a cluster.

## 5. Impact, risks, and what to watch

- **No manifest/deployment changes.** GitOps values, ArgoCD, and rollback are unaffected — same one-digest-per-service model.
- **Builds get slower.** Producing the arm64 variant under emulation adds a few minutes per build. This is expected, not a fault.
- **Every service Dockerfile must be arm64-buildable.** The main risk. If a Dockerfile bakes in an architecture-specific artifact — a downloaded CLI pinned to amd64, a CGO/native build, a hardcoded `GOARCH`/`--platform`, or an `apt`/`apk` package with no arm64 build — the arm64 build leg will fail. That is the intended safety behaviour (fail at build, not at runtime), but it means the **first** multi-arch build of each service should be watched, and any arch-pinned step in a Dockerfile fixed to be arch-neutral.
- **Interim alternative.** If any service can't yet build for arm64, the stop-gap is to keep it single-arch (amd64) and constrain its demo workload to amd64 nodes via a node selector on architecture. This is a holding measure, not the destination — a mixed fleet wants multi-arch images.

## 6. Verification

- **Migrate (done):** the multi-arch build succeeded, which means the per-arch entrypoint assertion passed on **both** amd64 and arm64 — proof the image now runs on either node. Repoint the demo migration Job to the multi-arch digest in §2 and re-run; it will schedule on any node.
- **Services (after the change):** for each service, the first multi-arch build either succeeds (publishes a dual-arch manifest) or fails loudly on an arch-pinned Dockerfile step. Confirm a couple of representative services pull and run on an arm64 node before treating the fleet as arch-portable.

---

**Bottom line:** the failure was an architecture mismatch, not a missing code fix. The estate is moving to multi-architecture images so one digest runs on both amd64 and arm64 nodes. The migrate image is fixed and its multi-arch build is in ECR (digest in §2); the identical change to the reusable service-build workflow in this repo fixes all services at once. Deployment mechanics don't change; the main thing to watch is that each service's Dockerfile can actually build for arm64.
