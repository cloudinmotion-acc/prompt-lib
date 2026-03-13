# Terraform Module Generation Prompt Pack

## Purpose & Usage

This prompt pack provides **repeatable, versioned instructions** for generating and validating Terraform AWS infrastructure modules while maintaining organizational consistency, security, and cost optimization.

### Where It's Used
- **Code generation**: Creating new Terraform modules (EKS, RDS, Bastion, etc.)
- **Code review**: Validating existing modules against standards
- **Refactoring**: Restructuring or consolidating modules
- **Dependency mapping**: Identifying upstream/downstream module requirements
- **Team onboarding**: Teaching new engineers the org's Terraform patterns

### Typical Workflow
1. Engineer runs `prompt_pack.yaml` system + developer + user prompt through LLM
2. LLM generates complete Terraform module files
3. Engineer reviews output against evaluation set (see eval/ folder)
4. Engineer makes minor adjustments (variable names, tags, etc.)
5. Engineer runs `terraform init && terraform validate && terraform plan`
6. Engineer applies changes and logs output schema for downstream modules
7. Engineer commits module and updates evaluation set with new real-world examples

---

## Version Management

### Semantic Versioning Rules

**MAJOR** (1.0.0 → 2.0.0):
- Breaking change to module input patterns (e.g., `platform_output` structure changes)
- Removal of a required constraint or pattern
- New mandatory organizational standard that existing modules violate

**MINOR** (1.0.0 → 1.1.0):
- New optional patterns or guidance
- Addition of new organizational constraints (e.g., "all RDS must be in Multi-AZ")
- Updated best practices or tool recommendations
- New evaluation categories

**PATCH** (1.0.0 → 1.0.1):
- Bug fixes in example code
- Clarifications in prompts or documentation
- Updated links or references
- Typo corrections

### How to Bump Version
1. Edit `version: "X.Y.Z"` in `prompt_pack.yaml`
2. Add entry to `CHANGELOG.md` with date and changes
3. Commit with message: `chore: bump prompt pack to vX.Y.Z`
4. Tag commit: `git tag prompts/terraform-module-generation/vX.Y.Z`
5. Push: `git push origin main --tags`

### Git Tracking
```bash
# View all prompt pack versions
git log --oneline prompts/terraform-module-generation/

# Checkout specific version
git show prompts/terraform-module-generation/v1.0.0:prompts/terraform-module-generation/prompt_pack.yaml

# Compare versions
git diff v1.0.0..v1.1.0 -- prompts/terraform-module-generation/
```

---

## Running Evaluations Locally

### Setup
```bash
# Install dependencies (if using evaluation runner)
pip install -r eval/requirements.txt  # (if Python-based)

# Or use manual evaluation (recommended for first runs)
```

### Manual Evaluation (Recommended for Teams)
1. **Prepare test case**: Pick a scenario from `eval/eval_set.jsonl`
   ```bash
   head -1 eval/eval_set.jsonl | jq .  # View first test
   ```

2. **Run prompt through LLM**:
   ```bash
   # Using Claude API
   cat prompts/terraform-module-generation/prompt_pack.yaml > /tmp/system.txt
   cat /tmp/eval_input.json > /tmp/user.txt
   
   # Feed to LLM provider's CLI or API
   ```

3. **Score output using rubric** (see `eval/eval_runner.md`):
   - Terraform validity: Does `terraform validate` pass? (yes/no)
   - Security: Any hardcoded secrets, missing encryption? (grade)
   - Dependency correctness: Correct upstream refs? (grade)
   - Naming consistency: Follows org patterns? (grade)
   - Documentation: Inline comments present? (grade)

4. **Log results**:
   ```bash
   echo '{"test_id":"rds-prod-001","timestamp":"2026-03-13T10:00Z","terraform_valid":true,"security_score":9,"dependency_score":10}' >> eval/results.jsonl
   ```

### Automated Testing (Future)
When you establish CI/CD:
```bash
# terraform-validate.sh
for module in eval/outputs/*.tf; do
  terraform -chdir=$(dirname $module) validate || echo "FAIL: $module"
done
```

---

## Rollback Instructions

If a prompt pack version introduces broken code or bad patterns:

### Quick Rollback
```bash
# Revert to previous known-good version
git checkout prompts/terraform-module-generation/v1.0.0 -- prompts/terraform-module-generation/

# Or revert entire commit
git revert <commit-hash>
```

### Longer-term Fix
1. Identify root cause of bad generation (ambiguous prompt, missing example, etc.)
2. Open PR to fix system_prompt or few_shot_examples
3. Test fix with eval_set scenarios
4. Release as PATCH version (or MINOR if significant change)
5. Communicate change to team with migration guide

---

## Common Failure Modes & Fixes

### Failure 1: "Generated module doesn't reference upstream outputs"
**Root cause**: User prompt didn't specify upstream dependencies clearly.
**Fix**: 
- In user_prompt_templates, explicitly list: `Upstream Dependencies: {{INIT_AVAILABLE}}, {{EKS_AVAILABLE}}`
- Add to few_shot_examples a case showing correct dependency wiring
- Update developer_prompt to emphasize "always validate depends_on chain"

**User workaround**:
```
Regenerate with explicit upstream context:
"Init module outputs: vpc_id, kms_key_arn, private_subnets, ssh_key_name
Ensure generated module reads these from var.initialization_output"
```

### Failure 2: "Generated code has hardcoded AWS account IDs or subnet IDs"
**Root cause**: Model hallucinating specific values instead of using variables/data sources.
**Fix**:
- Add to guardrails: "Never output sample values like subnet-12345; use {{PLACEHOLDER}}"
- Add to few_shot_examples a bad_example showing this mistake
- Increase strictness: lower temperature to 0.1 or use strict user_prompt_template

**User workaround**:
```
Explicitly state: "All subnet IDs must come from var.initialization_output.private_subnets"
In addition to dynamic variable refs, never hardcode specific AWS resource IDs.
```

### Failure 3: "Generated IAM policies too permissive (actions: *)"
**Root cause**: Model matching AWS Terraform examples which sometimes use wildcards for demo.
**Fix**:
- Add guardrail example showing correct ECR pull policy (specific actions, specific resources)
- Update system_prompt to emphasize: "IAM Least Privilege is NON-NEGOTIABLE"
- Add eval test case checking for wildcard actions (must fail)

**User workaround**:
```
Regenerate with: "IAM policies MUST specify exact actions (no wildcards).
Example: instead of ['ecs:*'], use ['ecs:DescribeTaskDefinition', 'ecs:DescribeTasks']"
```

### Failure 4: "Generated module misses required outputs"
**Root cause**: Model generates main.tf correctly but incomplete outputs.tf.
**Fix**:
- Add to few_shot_examples a complete end-to-end case with both technical_output and mpp_report
- Update user_prompt_templates to explicitly list: "Output Files Needed: main.tf, variables.tf, locals.tf, **outputs.tf**, versions.tf"
- Add eval test case for missing outputs

**User workaround**:
```
If outputs missing, regenerate with:
"Generate complete outputs.tf with two output blocks:
1. 'kubernetes_cluster_output' (technical, for downstream modules)
2. 'mpp_report' (user-facing map for platform UI)"
```

### Failure 5: "Environment-based logic missing (e.g., all environments get prod-sized instances)"
**Root cause**: Model not using var.platform_output.environment_type in computed locals.
**Fix**:
- Add to few_shot_examples a locals.tf example showing environment logic
- Update system_prompt with explicit rules: "node_group_desired_capacity = prod ? 5 : 3"
- Add eval test case checking for ternary operators on env_type

**User workaround**:
```
Regenerate with explicit env logic:
"Use var.platform_output.environment_type in locals to compute:
- instance sizes (prod: m5.xlarge, dev: t3.xlarge)
- node counts (prod: 5+, dev: 2-3)
- backup retention (prod: 30 days, dev: 7 days)"
```

---

## FAQ

**Q: Can I use this prompt pack with GPT-3.5 Turbo or open-source models?**
A: Yes. Performance may be lower (more hallucinations). Mitigate by:
- Use strict user_prompt_template instead of flexible
- Provide more few_shot_examples in context
- Lower temperature to 0.1
- Use smaller modules (start with simpler resources before complex ones)

**Q: How do I add a new constraint (e.g., "all RDS must be in Multi-AZ")?**
A: 
1. Add to system_prompt under "Environment-Based Logic"
2. Add example to few_shot_examples showing correct Multi-AZ config
3. Add test case to eval_set.jsonl checking Multi-AZ presence
4. Bump version to MINOR
5. Communicate change in CHANGELOG

**Q: Can I use this for non-AWS (Azure, GCP) modules?**
A: Not yet. This pack assumes AWS Terraform provider. For multi-cloud:
1. Create separate prompt pack for Azure (`prompt_pack_azure.yaml`)
2. Reference shared standards in a parent `prompt_pack_common.yaml`
3. Version them independently

**Q: How do I integrate this with CI/CD?**
A: See "Automated Testing (Future)" section. Example:
```yaml
# .github/workflows/prompt-validation.yml
- name: Validate Terraform via Prompt Pack
  run: |
    for test in prompts/terraform-module-generation/eval/eval_set.jsonl; do
      terraform-validate-against-pack.sh $test
    done
```
