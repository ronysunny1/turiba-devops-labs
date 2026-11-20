# Lab 10 · Build it from code with OpenTofu

GitHub user: ronysunny1 · Fallback chart used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste the password, the state file or the kubeconfig. -->

## 1 · The cluster and the app from code

```text
$ tofu plan                                    (Part 1: the end of the first plan)
Plan: 1 to add, 0 to change, 0 to destroy.
Changes to Outputs:
  + kube_context = (known after apply)

$ time tofu apply -auto-approve                (Part 2: from zero, the last lines and the time)
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
Outputs:
kube_context = "kind-course"
real 1m9.135s

$ tofu plan                                    (right after that apply)
No changes. Your infrastructure matches the configuration.

$ grep -c '"result"' terraform.tfstate
2
```

- 🔮 Before the first plan (Part 1): how many resources will it add, and which values will be `known after apply`? I predicted 1 resource (kind_cluster.course) would be added and cluster certificates, endpoint, kubeconfig, and kube_context would be (known after apply) because OpenTofu cannot know generated TLS certificates or the local API port until kind creates the cluster. What happened: Plan: 1 to add, 0 to change, 0 to destroy, and those exact computed cluster values showed (known after apply).
- 🔮 Before the second plan (Part 2): what will `tofu plan` say right after `tofu apply`? I predicted No changes. Your infrastructure matches the configuration. because all 5 resources in code had just been created and recorded in terraform.tfstate with zero drift. What happened: tofu plan refreshed all 5 resources and reported No changes.
- 🔮 Where is the database password now? I predicted the generated password is stored in plaintext inside infra/terraform.tfstate and base64-encoded in db-secret because OpenTofu persists generated random_password attributes in state so future plans know the current value. What happened: tofu state show masked sensitive data while grep -c returned 2 occurrences in terraform.tfstate.

## 2 · Change, drift, rebuild

```text
$ tofu apply -var api_replicas=2               (the plan: the symbol and the value that changes)
# helm_release.course_app will be updated in-place (~)
~ value = "3" -> "2"
Plan: 0 to add, 1 to change, 0 to destroy.

$ helm history course-app -n course
1  superseded  course-app-0.1.0  Install complete
2  deployed    course-app-0.1.0  Upgrade complete

$ tofu plan -var api_replicas=2                (after kubectl -n course delete secret db-secret)
# kubernetes_secret_v1.db will be created (+)
Plan: 1 to add, 0 to change, 0 to destroy.

$ tofu plan -var api_replicas=2                (after kubectl -n course scale deployment api --replicas=1)
# kubernetes_secret_v1.db will be created (+)
Plan: 1 to add, 0 to change, 0 to destroy.

$ tofu apply                                   (the plan, and how many API Pods run afterwards)
Plan: 1 to add, 1 to change, 0 to destroy.
Apply complete! Resources: 1 added, 1 changed, 0 destroyed.
api   3/3   3   3

$ time tofu destroy -auto-approve / time tofu apply -auto-approve      (the last lines and the times)
Destroy complete! Resources: 5 destroyed. (real 0m2.410s)
Apply complete! Resources: 5 added, 0 changed, 0 destroyed. (real 1m8.842s)

$ curl -s localhost:8080/api/todos             (after the rebuild)
[]
```

- 🔮 Step 1, `-var api_replicas=2`: which symbol, and which value changes? I predicted ~ (in-place update) on helm_release.course_app changing value = "3" -> "2" because api_replicas is passed into a Helm set block rather than an immutable attribute. What happened: ~ appeared on helm_release.course_app, creating Helm revision 2 and scaling api to 2/2 Pods.
- 🔮 Step 2, the deleted Secret and the scaled Deployment: what will each plan show? I predicted deleting db-secret would show Plan: 1 to add (+ kubernetes_secret_v1.db) while scaling api via kubectl would show no extra drift because OpenTofu tracks kubernetes_secret_v1.db directly but manages the Deployment through the helm_release resource. What happened: both plans reported only Plan: 1 to add, 0 to change, 0 to destroy (+ kubernetes_secret_v1.db).
- 🔮 Step 3, a plain `tofu apply`: what will the plan contain, and how many API Pods afterwards? I predicted Plan: 1 to add, 1 to change, 0 to destroy, leaving 3/3 API Pods afterwards because OpenTofu would recreate db-secret and reset api_replicas to its default of 3. What happened: db-secret was recreated, helm_release upgraded to 3 replicas, and 3/3 API Pods ran.
- 🔮 Step 4, after `tofu destroy` and `tofu apply`: is the todo still there? I predicted curl would return [] (not there) because tofu destroy deleted the entire kind cluster along with the PostgreSQL PersistentVolumeClaim. What happened: after rebuilding from zero, curl -s localhost:8080/api/todos returned [].

## 3 · What broke and how I found it

- 🔮 Before `tofu plan` on the teammate's branch, I expected this to go wrong first: a provider lockfile mismatch error before Kubernetes resources are diffed, because OpenTofu checks versions.tf against .terraform.lock.hcl first.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | versions.tf changed kind source to hashicorp/kind, missing from .terraform.lock.hcl | tofu plan -> Error: Inconsistent dependency lock file (provider registry.opentofu.org/hashicorp/kind: required by this configuration but no version is selected) | fa3489c (fix(infra): restore tehcyx/kind provider source) |
| 2 | main.tf pointed helm_release.course_app to ./../chart/course-app (missing s) | tofu plan -> Error: Error locating chart (Unable to locate chart ./../chart/course-app: path not found) | 284b9fa (fix(infra): correct course-app chart path) |
| 3 | main.tf hardcoded namespace = "courses" instead of referencing kubernetes_namespace_v1.course | tofu plan -> -/+ resource "helm_release" "course_app" (namespace = "course" -> "courses" # forces replacement) | 5bf8194 (fix(infra): reference course namespace resource in helm release) |

## 4 · In my own words

- Why `.terraform.lock.hcl` goes into Git and `terraform.tfstate` doesn't: .terraform.lock.hcl pins exact provider versions and checksums so every teammate and CI runner installs identical binaries, whereas terraform.tfstate contains live cluster certificates and plaintext secrets like random_password.db that must never be committed to Git.
- What `-/+ … # forces replacement` means, and what it would have cost me in Part 4: -/+ means OpenTofu cannot update in-place and will destroy and recreate the resource; applying that in Part 4 would have uninstalled course-app in namespace course (wiping its PostgreSQL PVC and todos) and failed or deployed an isolated release into courses.
- What I would lose by ending the day with `kind delete cluster` instead of `tofu destroy`: kind delete cluster removes the cluster container behind OpenTofu back while leaving stale entries in terraform.tfstate and local kubeconfig, causing future tofu plan/apply commands to fail until state is manually cleaned up.
