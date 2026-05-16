---
id: AgDR-0029
timestamp: 2026-05-17T00:00:00Z
agent: claude-opus-4-7
model: claude-opus-4-7[1m]
session: framework-packaging-discussion
trigger: user-prompt
status: proposed
category: architecture
---

# Framework Packaging & Distribution

> In the context of distributing ApexYard to new adopters and shipping updates to existing ones, facing the cost of `git pull upstream` merge conflicts on every framework-file customisation and the absence of version pinning / clean install boundaries, I am proposing **a layered install model** (read-only framework dir + adopter customisation overlay) to achieve conflict-free upgrades and pinnable versions, accepting a one-time breaking migration for every existing adopter.

> **Status: PROPOSED.** Operator (Ahmed) will comment on the PR with their pick. Decision section will be finalised post-comment.

## Context

ApexYard distributes today via the **fork-as-install** model:

- Adopter forks `me2resh/apexyard` on GitHub
- Clones the fork locally, sets `upstream` remote
- Runs `/setup` to fill in `onboarding.yaml`, registers projects in `apexyard.projects.yaml`
- Pulls updates with `/update` — which runs `git fetch upstream && git merge upstream/main` on a sync branch

This works and has shipped the framework to several adopters. The pain points that motivate this AgDR:

1. **Conflict surface grows with customisation.** Framework files (`.claude/settings.json`, role files in `roles/`, skill `SKILL.md`s, hook scripts) live in the adopter's git tree alongside their own customisations. Any local edit to a framework file becomes a permanent merge-conflict candidate on every upstream sync. Adopters who tweak hooks or settings pay this tax forever.

2. **No version pinning.** There is no mechanical way to declare "this fork is on apexyard@1.2.0". The release-cut model (AgDR-0007) tags releases on `main`, but the adopter is on whatever commit they last merged. A skill saying "requires framework ≥ #242" can only be checked against `git log`.

3. **No clean install boundary.** Framework-essential paths (`.claude/`, `roles/`, `workflows/`, `templates/`) and adopter-customisation paths (`onboarding.yaml`, `apexyard.projects.yaml`, `projects/<name>/`, custom-skills/, custom-handbooks/) are not separated at the filesystem level. There is no `apexyard uninstall`, no way to mechanically tell "what came from the framework vs what I wrote".

4. **No try-before-fork path.** Today's first-touch is "fork on GitHub and commit to it". There is no `npx apexyard init` or `brew install apexyard` shape for adopters who want to evaluate the framework on an existing project without committing to a fork.

5. **Customisation pattern is partial.** AgDR-0023 introduced `custom-templates/` (path-mirroring overrides for templates). #243 introduced `custom-skills/` and `custom-handbooks/`. These solve part of the conflict problem for one slice each, but adopter overrides on `.claude/settings.json`, hooks, agents, role files, or skill SKILL.mds still require editing the framework's checked-in file. The "override layer" exists but is incomplete.

The brand-visibility argument that drove the fork-as-install model (AgDR origin: post-v1 `.apexyard/` dotfile pattern was renamed away from to be visible) is real and worth preserving. Any new packaging model must keep ApexYard discoverable on the adopter's GitHub org.

## Options Considered

| # | Option | Pros | Cons | Migration cost |
|---|--------|------|------|----------------|
| 1 | **Status quo — fork-as-install** (no change) | Already shipping; brand-visible (fork on GitHub); one tool surface (`git`); zero migration cost | Conflict surface grows forever; no version pin; no install boundary; no try-before-fork; no clean uninstall | None |
| 2 | **Release-tarball delivery** (keep fork, swap merge for tar extract) | Smallest change with biggest immediate UX win — `/update` becomes "download `apexyard-1.3.0.tar.gz`, extract over fork"; eliminates merge conflicts on framework files for non-customising adopters | Adopters who DID customise framework files silently lose their edits on every `/update`; loss of `git log` for framework files inside the fork; still no version pin (no manifest); brand visibility preserved | Low — `/update` skill change + release-attachment workflow; no adopter migration |
| 3 | **Layered install** (`.apexyard/` read-only dir + overlay) | Conflict-free upgrades forever (framework dir is never edited by adopter); version-pinnable (manifest with version + checksums); clean install boundary; supports `apexyard uninstall`; supports try-before-fork (`apexyard init` in any repo) | Re-introduces the v1 dotfile model the project moved away from for brand reasons (mitigable: ship a `README.md` at fork root that names ApexYard, keep `apexyard.projects.yaml` at root); requires a new install tool (CLI shim) AND every existing adopter migrates; breaks the "git is the only tool" property of today's flow | **High** — breaking change for every existing adopter; needs migration skill (similar shape to `/split-portfolio`) + framework refactor to relocate `.claude/` and friends into `.apexyard/` |
| 4 | **npm package** (`npx apexyard init`, `npm update apexyard`) | Standard install pattern; version-pinnable via `package.json`; great try-before-fork story (`npx`); composes with existing JS/TS adopters | Forces Node + npm dependency on every adopter (currently optional, only `/process` lint needs Node); doesn't compose with non-JS adopters cleanly; awkward for adopters whose ops fork is not a Node project; brand visibility weaker than fork |
| 5 | **Homebrew formula** (`brew install apexyard`) | Familiar pattern for macOS adopters; version-pinnable; clean install/uninstall; nice CLI entry point | macOS / Linux-bias (Windows adopters get nothing); requires a separate distribution channel (`brew tap me2resh/apexyard`); doesn't solve "what about the fork?" — formula still has to install something somewhere; brand visibility lower than fork |
| 6 | **One-line install script** (`curl -fsSL apexyard.dev/install.sh \| bash`) | Trivial first-touch; works cross-platform; no package-manager dependency | Notoriously hard to do safely (`curl \| bash` is a security anti-pattern many orgs ban); doesn't solve any of the upgrade-conflict issues (it's only a first-install convenience); requires a domain + hosted script; brand visibility low |
| 7 | **Framework as git submodule** (adopter ops repo embeds `me2resh/apexyard` as a submodule) | Version-pinnable (submodule SHA); clean separation of framework vs adopter content; no new tooling | Git submodules are widely disliked; adopter UX is materially worse (`git submodule update` foot-guns); brand visibility decent (submodule shows on GitHub); requires every existing adopter to migrate | **High** — same migration cost as option 3 |

### Combination plays worth flagging

- **2 + 3 staged**: ship release-tarball delivery now (option 2) as a transition step, then move to layered install (option 3) once the tarball pipeline is established. Lowers the "big-bang migration" risk of option 3 in exchange for two refactors instead of one.
- **3 + 4 layered**: layered install IS the model; npm package is the *delivery channel* for the layered install (npm pulls `.apexyard/` contents). Buys both "standard install pattern" and "conflict-free upgrades" at the cost of forcing Node on every adopter.
- **3 + brand mitigation**: layered install + a generated `README.md` at fork root that names ApexYard, keeps the `apexyard.projects.yaml` registry at root (visible), and renames the `.apexyard/` install dir to a visible name like `apexyard/` (no leading dot). Preserves brand visibility while keeping the layered-install benefit.

## Recommendation

**Lean: option 3 (layered install), with the brand-mitigation pattern from "3 + brand mitigation" above.**

Rationale:

1. **The conflict surface is the root cause.** Options 4, 5, 6 are first-install conveniences; they don't fix the recurring upgrade-conflict tax adopters pay forever. Option 2 fixes upgrades but silently overwrites adopter framework-file edits — same root cause, masked.
2. **Version pinning matters for skill compatibility.** As the framework grows (#242 introduced v1 → v2 portfolio path resolution; #243 introduced custom-skills), skills increasingly declare "requires framework ≥ N". Without a pinnable version, adopters discover compatibility breaks at run time. A manifest with `framework_version: 1.3.0` makes this declarative.
3. **The brand-visibility concern that killed v1 dotfile is mitigable.** Use a visible install dir (`apexyard/` not `.apexyard/`), keep the registry at fork root, ship a fork-root `README.md` naming the framework. The brand argument was real in v1 but not load-bearing on the dotfile choice itself.
4. **Migration cost is one-time.** Option 3 is breaking, but every other option that materially fixes the upgrade-conflict issue is also breaking (option 7 is worse, option 2 silently mutates). One painful migration is preferable to a permanent tax. The framework has the `/split-portfolio` precedent for shipping breaking migrations with explicit operator-confirmation gates.

The trade I am explicitly NOT recommending: **doing nothing** (option 1). The pain is real and growing; punting accumulates conflict-resolution effort across every adopter forever.

## Decision

**To be filled in after operator (Ahmed) comments on the PR with their pick.**

Awaiting input on:

- Which option (or combination) to pursue
- If layered install (option 3): brand-visibility mitigation shape — visible `apexyard/` dir vs dotfile `.apexyard/`
- If staged (option 2 → option 3): timeline for the second hop, or commit to single-step
- Migration story: blocking `/update` until the adopter has migrated, vs. parallel-track support

## Consequences

(Will be expanded once the decision is recorded; placeholders below for the layered-install lean.)

If **option 3 (layered install)** is chosen:

- Every existing adopter must run a migration skill (provisional name `/migrate-to-layered`) before their next framework update
- The framework refactor moves `.claude/`, `roles/`, `workflows/`, `templates/` under a new install dir (`apexyard/` or `.apexyard/`)
- `CLAUDE.md` rewrites its `@.claude/rules/*.md` imports to point at the new install dir
- A new manifest file (`apexyard/manifest.json` or similar) records framework version + file checksums; `/update` becomes a manifest-driven file-replace
- `/update` no longer touches git; framework-version drift is reported from manifest mtime + version field, not `git fetch upstream`
- The `custom-templates/`, `custom-skills/`, `custom-handbooks/` overlay patterns generalise: an adopter can also overlay `.claude/settings.json`, individual hooks, individual rules — all from outside the install dir
- AgDR-0023 (custom-templates) and #243 (custom-skills) become the dominant customisation path, not an exception

If **option 2 (release-tarball)** is chosen instead:

- `/update` skill is rewritten to download + extract; existing fork structure preserved
- Adopters who customised framework files must be warned at `/update` time; opt-in protection (config-driven allow-list?) needs design
- No version pin; framework-version drift continues to read from `git log`

If **status quo** is chosen:

- Document the trade-off explicitly in `docs/multi-project.md` so adopters know the upgrade-conflict tax is intentional
- File a follow-on ticket to ship the partial mitigations (more `custom-*` overlay paths) instead

## Artifacts

- Tracker ticket: [me2resh/apexyard#265](https://github.com/me2resh/apexyard/issues/265)
- Related: [AgDR-0007](AgDR-0007-release-cut-branch-model.md) (release-cut branch model — the version source this AgDR would pin against)
- Related: [AgDR-0021](AgDR-0021-split-portfolio-v2-path-resolution.md) (split-portfolio path resolution — the customisation overlay pattern this AgDR generalises)
- Related: [AgDR-0023](AgDR-0023-custom-templates-override-semantics.md) (custom-templates override — the path-mirroring shape this AgDR scales up)
- PR: (will be filled in once opened)
