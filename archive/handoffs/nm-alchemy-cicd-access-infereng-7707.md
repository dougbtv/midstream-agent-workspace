# Agent Handoff: INFERENG-7707 — Grant nm-cicd write access via nm-alchemy

## What you're doing

Add **write access** to the [nm-cicd](https://github.com/neuralmagic/nm-cicd) repo for the same set of users who currently have access to [nm-vllm-omni-ent](https://github.com/neuralmagic/nm-vllm-omni-ent), using **nm-alchemy** (our GitHub permissions management tool).

Write access is needed so team members can trigger GitHub Actions workflows (`workflow_dispatch`) on nm-cicd — specifically the vLLM-Omni build and test workflows.

## JIRA

- **This story:** [INFERENG-7707](https://redhat.atlassian.net/browse/INFERENG-7707) — In Progress, assigned to Doug
- **Epic:** [INFERENG-6288](https://redhat.atlassian.net/browse/INFERENG-6288) — vLLM-Omni Midstream: Initial Build

## What is nm-alchemy?

nm-alchemy is a repo at https://github.com/neuralmagic/nm-alchemy that manages GitHub permissions declaratively. It contains config files (likely YAML or TOML) that define which users/teams get what access level on which repos. Changes are made via PR → merge → permissions sync.

## Steps

1. **Clone or navigate to nm-alchemy:**
   ```bash
   gh repo clone neuralmagic/nm-alchemy /tmp/nm-alchemy
   # or if it's already local, find it
   ```

2. **Find the config that grants access to `nm-vllm-omni-ent`:**
   ```bash
   cd /tmp/nm-alchemy
   grep -r "nm-vllm-omni-ent" --include="*.yml" --include="*.yaml" --include="*.toml" --include="*.json" .
   ```
   This will show you which team/group definition includes the omni repo.

3. **Add `nm-cicd` to the same group** with `write` (or `push`) access. The exact syntax depends on nm-alchemy's config format — follow the pattern used for nm-vllm-omni-ent.

4. **Open a PR** on nm-alchemy with the change. Title suggestion: "Add nm-cicd write access for vLLM-Omni team"

5. **Verify** after merge — confirm a team member can dispatch a workflow:
   ```bash
   gh workflow run test-vllm-omni.yml --repo neuralmagic/nm-cicd \
     --ref vllm-omni-build \
     -f vllm_image=quay.io/vllm/automation-vllm-omni:cuda-26785885017 \
     -f model=Tongyi-MAI/Z-Image-Turbo \
     -f label=k8s-a100-solo
   ```

## What "write access" means here

GitHub requires **write** permission on a repo to use `workflow_dispatch` (trigger workflows manually via API or `gh workflow run`). Read-only access is not enough. The access level needed is `write` (also called `push` in some contexts), NOT `admin`.

## Repos involved

| Repo | Purpose |
|------|---------|
| [nm-alchemy](https://github.com/neuralmagic/nm-alchemy) | GitHub permissions config — this is where you make the change |
| [nm-cicd](https://github.com/neuralmagic/nm-cicd) | Target repo — needs write access granted |
| [nm-vllm-omni-ent](https://github.com/neuralmagic/nm-vllm-omni-ent) | Reference — find who already has access here, give them same on nm-cicd |

## JIRA CLI

```bash
# Comment on the story
cat > /tmp/jira-comment.md << 'EOF'
### Your update here
EOF
jira issue comment add INFERENG-7707 -T /tmp/jira-comment.md --no-input

# Close when done
jira issue move INFERENG-7707 "Closed"
```

## Definition of done

- [ ] nm-alchemy PR opened adding nm-cicd write access for vLLM-Omni team members
- [ ] PR merged
- [ ] Verified: at least one team member can `gh workflow run` on nm-cicd
- [ ] JIRA updated and closed
