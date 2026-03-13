# Evaluation Procedure: Terraform Module Generation Prompt Pack

## Quick Start

1. Pick a test case from `eval_set.jsonl`
2. Run the prompt pack system + user prompt through LLM
3. Score using rubric below
4. Compare against expected_traits, must_include, must_not_include
5. Log results in `results.jsonl`

---

## Scoring Rubric

### Category 1: Terraform Syntax & Validity (25 points)

**Criteria:**
- Code must pass `terraform validate` (no syntax errors)
- HCL2 formatting is correct (no stray `@{}`, quotes, etc.)
- All variable references resolve (no undefined vars)

**Scoring:**
- ✅ 25pts: Passes `terraform validate` cleanly
- ⚠️ 15pts: Minor syntax issues (fixable in <2 min)
- ❌ 0pts: Fails validation; unfixable without rewrite

**How to Test:**
```bash
# Copy generated .tf files into a temp directory with versions.tf
terraform init
terraform validate
```

---

### Category 2: Security & Compliance (25 points)

**Criteria:**
- No hardcoded secrets, passwords, or API keys
- All encrypted resources specify KMS key
- IAM policies use specific actions, not wildcards
- All egress to 0.0.0.0/0 justified with inline comment
- Secrets Manager used for sensitive data (prod: 30-day recovery, dev: 7-day)
- tfsec ignores justified (not suppressing real issues)

**Scoring:**
- ✅ 25pts: All 6 criteria met
- ⚠️ 15pts: Minor gaps (e.g., missing one tfsec comment)
- ⚠️ 10pts: Moderate gaps (e.g., 1 hardcoded value, or wildcard IAM action)
- ❌ 0pts: Major security hole (exposed secret, overly permissive IAM, etc.)

**How to Test:**
```bash
# Check for hardcoded secrets
grep -r "password.*=" *.tf | grep -v "random_password\|aws_secretsmanager"

# Check for wildcard IAM actions
grep -E '"[*]"|actions.*\[.*"\*"' *.tf

# Check for 0.0.0.0/0 without comments
grep -B2 '0\.0\.0\.0/0' *.tf | grep -v '#tfsec:ignore' | grep egress
```

---

### Category 3: Dependency & Module Architecture (20 points)

**Criteria:**
- Correctly references upstream module outputs (var.initialization_output, etc.)
- No hardcoded AWS resource IDs (subnet IDs, account IDs, etc.)
- Proper dependency ordering (depends_on, explicit upstream refs)
- Outputs include both technical (for downstream) and mpp_report (for users)
- Naming follows org convention: `${system_name}-${resource}-${environment}`

**Scoring:**
- ✅ 20pts: All 5 criteria met
- ⚠️ 12pts: Minor dependency issue (e.g., missing 1 depends_on or hardcoded subnet)
- ⚠️ 8pts: Moderate issue (e.g., improper output structure or naming inconsistency)
- ❌ 0pts: Broken dependency chain (e.g., hardcoded values breaking portability)

**How to Test:**
```bash
# Check for hardcoded subnet IDs
grep -E "subnet-[a-z0-9]+|10\.0\." *.tf

# Check for hardcoded account IDs
grep -E "arn:aws:[^:]*:[^:]*:[0-9]{12}" *.tf

# Verify outputs structure
grep -A5 'output "' outputs.tf | grep -E '(technical|mpp_report)'
```

---

### Category 4: Environment-Based Logic & Cost (15 points)

**Criteria:**
- Uses `var.platform_output.environment_type` to determine:
  - Instance sizes (prod: larger, dev: smaller)
  - Node counts (prod: 5+, dev: 2-3)
  - Backup retention (prod: 30+ days, dev: 7 days)
  - Multi-AZ (prod: true, dev: false) *where applicable*
  - Deletion protection (prod: true, dev: false) *where applicable*
- Cost implications noted or justified in comments

**Scoring:**
- ✅ 15pts: All 5 environment-based decisions present
- ⚠️ 10pts: 3-4 decisions present; minor gaps
- ⚠️ 5pts: 1-2 decisions present; significant gaps
- ❌ 0pts: One-size-fits-all (no env logic); or prod config forced in dev

**How to Test:**
```bash
# Check for ternary operators on environment_type
grep -r "environment_type.*?" *.tf

# Count distinct env-based decisions
echo "Decisions found:" && grep -c "prod.*dev\|production.*development" *.tf
```

---

### Category 5: Documentation & Code Quality (15 points)

**Criteria:**
- Inline comments explain **why**, not just **what**
- Variables have meaningful descriptions
- Locals have clear naming (e.g., `common_tags`, `resource_prefix`)
- README or usage guide present
- Validation rules on sensitive variables (e.g., instance type, storage size)

**Scoring:**
- ✅ 15pts: All criteria met; code is self-documenting
- ⚠️ 10pts: Comments present but sparse; some variables lack descriptions
- ⚠️ 5pts: Minimal comments; documentation unclear
- ❌ 0pts: No comments; unreadable variable names; no validation

**How to Test:**
```bash
# Count comments
grep -c "^[[:space:]]*#" *.tf

# Check variable descriptions (should have at least 80% of variables documented)
grep -c 'description =' variables.tf
```

---

## Quick Scoring Template

```
Test ID: {{TEST_ID}}
Evaluator: {{NAME}}
Date: {{DATE}}

| Category | Score | Notes |
|----------|-------|-------|
| Syntax & Validity | __/25 | |
| Security & Compliance | __/25 | |
| Dependency & Architecture | __/20 | |
| Environment & Cost | __/15 | |
| Documentation | __/15 | |
| **TOTAL** | __/100 | |

Pass/Fail: [✅ PASS / ⚠️ CONDITIONAL / ❌ FAIL]
Conditional Note: [If applicable, what needs fixing?]

Issues Found:
1. [Issue]: [Severity]. [How to fix].

Recommendations:
1. [Suggestion for improvement]
```

---

## Comparing Prompt Versions

### Setup: Version A vs Version B

```bash
# Generate code with prompt_pack v1.0.0
# Save output to /tmp/v1_output.tf

# Generate same test case with prompt_pack v1.1.0
# Save output to /tmp/v2_output.tf

# Compare scores
diff <(cat /tmp/v1_results.txt | jq '.score') \
     <(cat /tmp/v2_results.txt | jq '.score')
```

### Metrics to Compare

1. **Syntax Pass Rate**: % of test cases that pass `terraform validate`
   - Target: 95%+ (1 or fewer failures per 20 tests)

2. **Security Score**: Average security category score across test set
   - Target: 23/25 (92%+)
   - Regression: If v1.1.0 scores < v1.0.0 by 3+ points, investigate

3. **Dependency Correctness**: % of tests with correct upstream refs
   - Target: 98%+

4. **Documentation Quality**: % of variables with descriptions
   - Target: 90%+

5. **Cost Alignment**: % of modules with proper env-based logic
   - Target: 90%+

### Regression Test (On Every Version Bump)

```bash
# Run all tests in eval_set.jsonl against new prompt version
for test in $(jq -r '.id' eval/eval_set.jsonl); do
  run_prompt $test
  score=$(evaluate_output | jq '.total_score')
  if [ $score -lt 85 ]; then
    echo "REGRESSION: $test scored $score (was baseline X)"
  fi
done
```

---

## Acceptance Thresholds

### For Code Generation (User Prompt → Output)
- **Must Pass**: Terraform syntax valid (terraform validate ✅)
- **Must Include**: All must_include items found in code
- **Must Not Include**: None of must_not_include items present
- **Minimum Score**: 80/100 to merge without human review
- **Conditional**: 70-79 score → requires human review + fixes
- **Rejection**: <70 score or fails "Must Pass" → regenerate or revert prompt version

### For Code Review / Validation (User Submits Code → LLM Reviews)
- **Identifies Issues**: All major security/dependency issues flagged
- **Suggests Fixes**: Recommendations are actionable and correct
- **Accuracy**: Review aligns with actual issues (no false positives)
- **Minimum Score**: 75/100 for review reliability

### For Cross-Module Validation
- **Dependency Integrity**: No circular deps, all outputs match upstream inputs
- **Type Consistency**: Variable types match exported types
- **Naming Alignment**: No mismatches in output names
- **Minimum Score**: 90/100 (strict)

---

## Logging Results

```bash
# Append to results.jsonl after each eval
cat >> eval/results.jsonl <<EOF
{
  "test_id": "rds-pg-dev-001",
  "evaluator": "akshaya",
  "timestamp": "2026-03-13T10:30:00Z",
  "prompt_version": "1.0.0",
  "model_used": "claude-3.5-sonnet",
  "scores": {
    "syntax_validity": 25,
    "security_compliance": 23,
    "dependency_architecture": 18,
    "environment_logic": 12,
    "documentation": 14,
    "total": 92
  },
  "pass_fail": "PASS",
  "issues": [
    {
      "severity": "warning",
      "description": "Missing one tfsec:ignore comment on HTTP egress rule",
      "fix": "Add #tfsec:ignore:aws-vpc-no-public-egress-sgr to line 23"
    }
  ]
}
EOF

# Analyze trends
jq '[.scores.total] | add / length' eval/results.jsonl  # Average score
jq '[select(.pass_fail == "PASS")] | length' eval/results.jsonl  # Pass count
```

---

## Tips for Best Results

1. **Run multiple times**: Different temps/models may yield different outputs. Average across 3 runs for reliability.
2. **Control variables**: Keep test case, model, and temperature constant when comparing versions.
3. **Log everything**: Save outputs and scores; enables debugging of failures.
4. **Isolate changes**: When bumping prompt version, change one thing (e.g., add one example) and measure impact.
5. **Regression testing**: Always run baseline tests against new prompt before committing.
