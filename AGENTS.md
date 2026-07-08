# AGENTS.md

## What is this repository

This repository contains Kustomize-compatible templates and Kubernetes custom
resources for the validated architecture (VA) and deployment topology (DT)
workstreams of RHOSO (Red Hat OpenStack Services on OpenShift).

- **Validated Architectures (VAs)** represent production-style deployment
  environments.
- **Deployment Topologies (DTs)** are for test environments only.

The upstream repository lives at
`https://github.com/openstack-k8s-operators/architecture`.

## Tech stack

- **Kustomize** 5.0.1+ (or OpenShift CLI `oc` 4.14+).
- **YAML** for all templates, CRs, values, and automation definitions.
- **MkDocs** (Material theme) for documentation.
- **Yamale** for automation schema validation.
- **Python** scripts for Zuul job generation and schema validation.
- **GitHub Actions** and **Zuul** for CI.

## Repository layout

| Path | Description |
|---|---|
| `lib/` | Base templates common to all VAs and DTs. |
| `lib/control-plane/` | Control plane components (base, dns, storage, ovn-bridge, etc.). |
| `lib/dataplane/` | Data plane components (nodesets, deployments). |
| `lib/networking/` | Network configurations (metallb, nad, netconfig, nncp). |
| `va/` | VA-specific templates. |
| `dt/` | DT-specific templates. |
| `examples/va/` | User-environment VA templates. Users are expected to modify these. |
| `examples/dt/` | User-environment DT templates. Users are expected to modify these. |
| `automation/` | Stage definitions (YAML) for each VA/DT: paths to CR kustomizations, values, and validation commands. |
| `docs/` | MkDocs sources. Contributing guides under `docs/contributing/`. |
| `zuul.d/` | Zuul job definitions. **Some files are generated -- see below.** |
| `.ci/` | Yamale schema (`automation-schema.yaml`) and validation script. |

## Critical rules

### Kustomize patterns

Templates use the kustomize components and overlays pattern for reusability.
Values-based customization is done through YAML files. Remote builds via
GitHub URLs are supported for referencing specific branches or tags.

### Three-layer structure

Templates are organized in three layers:

1. `lib/` -- shared base templates common to all VAs and DTs.
2. `va/`/`dt/` -- specific templates for a given VA or DT.
3. `examples/va/`/`examples/dt/` -- user-environment templates expected to be
   customized for a specific deployment.

When editing, understand which layer you are in. Changes to `lib/` affect
all VAs and DTs. Changes to `examples/` affect only the user-facing
configuration.

When a `lib/` component changes, trace its consumers by searching for it
in `va/` and `dt/` kustomizations, then in `examples/` kustomizations that
reference those. Build every affected stage to confirm nothing is broken.

### VA vs DT distinction

This is a hard rule: DTs are **test-only** and must not be presented as
production guidance. VAs are **production-oriented** and must represent
real-world deployment patterns.

### Automation stage format

Files under `automation/vars/` define deployment stages. Each stage specifies:
- `path` -- path to the CR kustomization
- `values` -- list of value files to apply
- `wait_conditions` -- `oc`/`kubectl` wait commands to validate stage completion
- Optional `pre_stage_run`/`post_stage_run` hooks

These files are validated against the Yamale schema at
`.ci/automation-schema.yaml`. If you change the automation structure, update
the schema as well.

#### Standard stage ordering

Stages in `automation/vars/<name>.yaml` follow this conventional order:

1. `nncp-configuration` -- Node Network Configuration Policies (may include
   pre-stage node labeling). Timeout: **5m**.
2. `network-configuration` -- MetalLB, broader networking (wait for MetalLB
   speaker pods).
3. `control-plane` -- OpenStackControlPlane CR. Timeout: **60m**.
4. `edpm-networker-nodeset` -- Networker dataplane nodeset (if applicable).
   Timeout: **10m**.
5. `edpm-networker-deployment` -- Networker deployment (if applicable).
   Timeout: **40m**.
6. `edpm-nodeset` -- Compute dataplane nodeset (wait for SetupReady).
   Timeout: **10m**.
7. `edpm-deployment` -- Compute dataplane deployment. Timeout: **60m**.

Timeouts shown are the most common values. Actual timeouts vary by
topology — check the closest existing automation vars file rather than
copying these defaults.

Not all stages are required. Simple topologies may skip networker stages.
Ceph-based topologies split dataplane into pre-ceph and post-ceph stages.

#### Stage definition example

Each automation vars file wraps stages under `vas: <topology-name>:` (the
repo uses `vas:` as the top-level key for both VAs and DTs):

```yaml
vas:
  <topology-name>:
    stages:
      - name: nncp-configuration
        path: examples/dt/<topology>/control-plane/networking/nncp
        values:
          - name: network-values
            src_file: values.yaml
        build_output: nncp.yaml
        wait_conditions:
          - >-
            oc -n openstack wait nncp
            -l osp/nncm-config-type=standard
            --for jsonpath='{.status.conditions[0].reason}'=SuccessfullyConfigured
            --timeout=5m
```

`build_output` -- filename for the kustomize build output artifact.
ci-framework's `kustomize_deploy` role runs `kustomize build` on the stage's
`path` and writes the result to this file, then applies it with `oc apply`.
Every stage should specify one; the name is freeform but conventionally
reflects the stage (e.g., `nncp.yaml`, `control-plane.yaml`, `edpm.yaml`).

The `values` field uses objects with `name` (ConfigMap name) and `src_file`
(filename relative to `path`).

#### `pre_stage_run` / `post_stage_run` hook format

The optional `pre_stage_run` and `post_stage_run` fields are siblings of
`name`, `path`, and `values` within a stage list item. They accept a list
of structured hook objects. Two hook types are supported:

Both hook types share `name` and `type`. The remaining fields are
type-specific — do not mix them:

- **`type: cr`** fields: `kind`, `resource_name`, `state`, `definition`
  (and optionally `api_version`, `namespace` for non-core resources).
- **`type: playbook`** fields: `source`, `inventory`.

**`type: cr`** -- patch or create a Kubernetes resource:

```yaml
      - name: nncp-configuration
        pre_stage_run:
          - name: "Apply cinder-lvm label"
            type: cr
            kind: Node
            resource_name: master-0
            state: patched
            definition:
              metadata:
                labels:
                  openstack.org/cinder-lvm: ""
        path: examples/dt/<topology>/control-plane/networking/nncp
```

**`type: playbook`** -- run an Ansible playbook (used for Ceph deployment
and similar operations that require orchestration beyond `oc` commands):

```yaml
        post_stage_run:
          - name: "Deploy Ceph"
            type: playbook
            source: "../../hooks/playbooks/ceph.yml"
            inventory: "${HOME}/ci-framework-data/artifacts/zuul_inventory.yml"
```

The `source` path is resolved by ci-framework at runtime; paths with
`../../` are relative to the automation vars file location, while bare
filenames are resolved from ci-framework's playbook search paths. The
`inventory` path may use environment variables.

### Generated files -- do not hand-edit

The following files are **generated** by `create-zuul-jobs.py`:

- `zuul.d/validations.yaml`
- `zuul.d/projects.yaml`

CI verifies that committed files match the output of `create-zuul-jobs.py`.
If you hand-edit these files, CI will fail.

## Validation and testing

Run validation in this order before committing:

1. `yamllint -c .yamllint.yml -s .` — lint all YAML.
2. `./test-kustomizations.sh examples/` — verify Kustomize builds.
3. `yamale -s .ci/automation-schema.yaml automation/vars/` — validate automation
   stage files against schema.
4. `python3 .ci/validate-schema-paths.py` — verify paths in stage files exist.
5. `./create-zuul-jobs.py` — regenerate Zuul jobs if `automation/vars/` changed,
   then confirm `zuul.d/validations.yaml` and `zuul.d/projects.yaml` match.

### Inspecting what a change deploys

`./test-kustomizations.sh` confirms builds succeed but does not show you
what gets deployed. To inspect the full Kubernetes CRs a kustomization
produces:

`kustomize build examples/va/<name>/<stage> > /tmp/out.yaml`

For a VA or DT with multiple stages, build each path listed under
`stages[*].path` in `automation/vars/<name>.yaml` in order. Use `grep`
to confirm new fields appear in the output and removed fields are absent.

### CI checks (GitHub Actions)

- **yaml-lint**: `yamllint` on all YAML.
- **test-kustomize**: `kustomize build` on `examples/`.
- **test-zuul**: Regenerates Zuul jobs and verifies checksum matches committed files.
- **automation-schema**: Yamale validation on `automation/vars/`, then `validate-schema-paths.py`.

### PR validation

PRs are expected to provide proof of validation. The recommended approach is
to use CI-Framework's reproducer to deploy the affected VA/DT. See
`docs/contributing/pull-request-testing.md` for step-by-step instructions.

## YAML

All YAML files must pass `yamllint`. Common causes of failure:

- **Missing end-of-file newline** — every YAML file must end with a newline.
- **Trailing whitespace** — no spaces or tabs at the end of any line.
  Check with `grep -n " $" file.yaml` before committing.
- **Indentation** — two spaces per level throughout.
- **Comments inside literal block scalars** — avoid inline comments inside
  `|` or `|-` blocks; they are treated as content, not comments.

## Documentation

Built with MkDocs (Material theme). Config in `mkdocs.yml`.

```
pip install -r docs/doc_requirements.txt
mkdocs serve
```

Contributing guides are under `docs/contributing/`:
- `documentation.md` -- how to add and structure docs
- `pull-request-testing.md` -- how to validate PRs with the reproducer
- `cherry-picking.md` -- backport process

## Review process

Governance is defined in the `OWNERS` file (Kubernetes-style approvers and
reviewers).

## Commit conventions

- **Title**: Short, descriptive summary of the change.
- **Body**: Describe **why** the change was made, not just what changed.
- **AI attribution**: Use `Co-Authored-By:` for substantial AI-generated code,
  `Assisted-By:` for minor AI help.
- **Ticket references**: Link Jira cards in the commit message body:
  `Closes: ANVIL-123` (resolves the ticket) or
  `Related-Issue: #OSPRH-12345` (related but does not close).
- **Cross-repo dependencies**: When a change depends on an unmerged PR/MR in
  another repository, add `Depends-On: <PR-or-MR-URL>` in the PR/MR
  description. Zuul uses this to test the changes together.

### Commit strategy

To keep a clean git history, prefer a single commit per feature or fix:

1. Create the initial commit normally.
2. For subsequent changes on the same branch, amend the existing commit
   (`git commit --amend`) instead of creating new ones.
3. After amending, use `git push --force` to update the remote branch.

Never push directly to `main` — it is a protected branch. Always work on
a feature branch. Force pushing is only appropriate for **solo feature
branches**, never for `main` or shared branches.

## Branch workflow

- The default branch is `main`.
- Feature work happens on topic branches.
- PRs target `main` unless otherwise specified.

## Documentation first

Before searching the web or relying on general knowledge, check local
documentation:
- `docs/contributing/` — PR testing, cherry-picking, and documentation guides.

## Confirm before acting

Before performing expensive or broad-impact operations, confirm with the
user first:

- Modifying `lib/` — changes affect **all** VAs and DTs.
- Changing the Yamale schema (`.ci/automation-schema.yaml`) — breaks
  validation for all automation stage files if done incorrectly.
- Cross-repo changes that require coordinated updates in `ci-framework`
  scenario files or `ci-framework-jobs` variable packs.

## Relationship to ci-framework

Though CI-Framework consumes this repository, it provides example CRs and
instructions on how to use them. This repository is useful to people who
do not use ci-framework since they can generate CRs for their deployment.

CI-Framework consumes this repository to deploy VAs and DTs. The
`kustomize_deploy` role reads architecture definitions and applies them
stage by stage. Uni jobs in `ci-framework-jobs` reference specific
architecture scenarios (e.g. `uni01alpha`, `uni03gamma`) which map to
automation files in this repo.

When making changes here, consider whether corresponding updates are needed
in the ci-framework scenario files or the ci-framework-jobs variable packs.

To test a scenario locally with CI-Framework:

```bash
git clone https://github.com/openstack-k8s-operators/ci-framework.git
cd ci-framework
make run_ctx_architecture_test \
    SCENARIO_NAME=your_scenario \
    ARCH_REPO=../architecture \
    NET_ENV_FILE=./ci/playbooks/files/networking-env-definition.yml
```
