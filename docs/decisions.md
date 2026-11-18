# Key platform decisions

The main infrastructure decisions behind the platform. Each entry gives the decision, why it was made, and what it costs.

## 1. Three repositories: infra, CI and CD

**Decision.** The project is split into three repositories. The infra repo builds the cloud with Terraform. The CI repo holds the application and its pipeline. The CD repo holds the desired state of the cluster.

**Why.** Each repo has one job and one set of permissions. Application developers don't need access to the cloud infrastructure. The pipeline only needs to write to the CD repo, never to the cluster. Every change to what is running shows up as a commit in one place.

**Trade-off.** A release touches two repositories (CI builds it, then CD records it), and there are three sets of access to manage instead of one.

## 2. Terraform workspaces instead of a folder per environment

**Decision.** One Terraform code base builds dev, stage and prod. The workspace name picks the environment, and a settings file (`dev.tfvars`, and so on) sets the sizes.

**Why.** All three environments run the same code, so what was tested in dev is what runs in prod. There is one copy to maintain instead of three folders that slowly drift apart. Each workspace has its own state, so destroying dev can't touch prod. Adding an environment means adding a workspace and a settings file.

**Trade-off.** It's possible to apply to the wrong workspace. Guardrails reduce the risk:

- The code rejects any workspace other than dev, stage or prod, including Terraform's built-in `default`.
- Every resource name includes the environment.
- The prod database has deletion protection.

If environments ever needed structurally different resources, not just different sizes, separate modules would be the better fit.

## 3. Pull-based delivery with Argo CD

**Decision.** Argo CD runs inside each cluster and pulls changes from the CD repo. The pipeline never deploys to the cluster directly.

**Why.**

- The cluster is private, and the pipeline doesn't need credentials or network access to it.
- Argo CD continuously makes the cluster match Git. It also undoes manual changes (self-heal) and removes resources that were deleted from Git (prune).
- Rolling back means reverting a commit.

**Trade-off.** Argo CD needs read access to the private CD repo. That is a one-time token per cluster, which has to be stored and rotated.

## 4. A LoadBalancer Service now, Ingress later

**Decision.** The frontend is exposed through a Kubernetes LoadBalancer Service. Ingress is designed in but not built.

**Why.** A LoadBalancer is the simplest way to get one public entry point, and it was enough to prove the platform end to end. Ingress depends on a business decision that hasn't been made yet: whether the app is public or internal, and which team owns the entry point. See the [roadmap](roadmap.md).

**Trade-off.** There's no HTTPS certificate, web application firewall or path-based routing yet.

## 5. Cloud SQL on a private IP only

**Decision.** The PostgreSQL database runs on Cloud SQL with no public address. The backend reaches it over a private connection inside the VPC.

**Why.** The database is never exposed to the internet, which removes a whole class of attacks. Google manages patching and backups.

**Trade-off.** Admin access to the database has to go through the private network, and teardown needs extra steps to remove the private network connection.

## 6. A separate `shared/` root with no destroy path

**Decision.** Resources used by every environment live in their own Terraform root, `shared/`, with one state file:

- the container registry
- the pipeline's permission to push images
- the backup bucket

There is no teardown procedure for this root.

**Why.** Environments come and go to save money, but the registry and its images must survive. Keeping shared resources in their own state means destroying an environment can't delete them. The remote state bucket itself is created by a separate `bootstrap/` root, run once by hand.

**Trade-off.** One more root to run in the right order. `shared/` must exist before any environment, and must never be torn down while an environment still uses it.
