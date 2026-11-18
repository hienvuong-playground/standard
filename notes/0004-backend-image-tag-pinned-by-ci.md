# 0004 – Backend image tag pinned by CI via auto-merged PR

## Context

The backend Deployment used `backend:latest` with `imagePullPolicy: Always`. CI pushed new images, but nothing in Git changed, so Flux had nothing to apply and pods never rolled to the new version.

Wanted: every build is deployed automatically, and every deploy leaves a PR behind as history (no review, merged automatically).

## Decision

**CI writes the tag into Git.** After pushing the image, a new `update-manifest` job in `backend/.github/workflows/build-push.yml`:

1. runs `kustomize edit set image` on `deploy/kustomization.yaml` with the tag just pushed (`sha-<first 7 chars of commit>`);
2. commits to a branch `deploy/backend-<tag>`, opens a PR and merges it with `gh pr merge --squash --delete-branch`.

Flux sees the new tag on `main` and rolls the Deployment. The manifest has no tag anymore (`deploy/deployment.yaml`); the tag lives only in the `images:` block of `deploy/kustomization.yaml`.

Details worth remembering:

- Workflow-level `concurrency` (no cancel) keeps runs in push order, so an older build can't overwrite a newer tag and two PRs never edit the same line at once.
- `deploy/kustomization.yaml` is formatted the way `kustomize edit` writes it (unindented lists). Otherwise every deploy PR rewrites the whole file instead of one line.
- The PR step is rerun-safe: force-push the bot branch, and only create the PR if none exists. The first run failed exactly here (branch left behind by an attempt that couldn't create the PR).
- No build loop: `deploy/` is outside the paths filter, and pushes by `GITHUB_TOKEN` don't trigger workflows.
- Tag over digest: the sha tag is unique per commit and readable in the PR. Only a manual re-run of the same commit overwrites it.

Rejected: **Flux Image Automation** (image-reflector + image-automation controllers). It is the "official" Flux way, but here it means enabling two extra controllers on the AKS extension, workload identity + AcrPull for the reflector, a write deploy key, and a scanning loop, only to rediscover a tag CI already knows. It pays off when manifests live in a separate repo CI must not write to, when several environments follow different tag policies, or for images you don't build. None of that applies.

## Consequences

- Needs "Allow GitHub Actions to create and approve pull requests" at org level and in the `backend` and `frontend` repos (frontend uses the same flow since 0011). Without it `gh pr create` fails. Originally set by hand; now managed by `infra/init` (0015). `main` must also have no branch protection that requires a review, or `gh pr merge` fails.
- Rollback = revert the deploy PR. `acr-cleanup.yml` keeps the 20 newest backend tags, so only builds within that window can be rolled back to; older ones would end in `ImagePullBackOff`.
- `:latest` is still pushed but no longer used by the deployment.
- Verified: PR #1 "Deploy backend sha-dd3d2e3" was created and merged by the workflow, and the cluster Deployment and pod run `backend:sha-dd3d2e3`.

## References

- https://argo-cd.readthedocs.io/en/stable/user-guide/ci_automation/
- https://fluxcd.io/flux/guides/image-update/
- https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/images/
- https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs
