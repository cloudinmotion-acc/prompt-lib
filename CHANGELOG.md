# Changelog: Terraform Module Generation Prompt Pack

All notable changes to the prompt pack are documented here.
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.0.0] - 2026-03-13

### Added
- **System Prompt**: Comprehensive Terraform architecture guidelines covering:
  - Organizational naming conventions (system_name-resource-environment)
  - Module input patterns (platform_output, initialization_output)
  - Locals patterns (common_tags, resource_prefix, env-based logic)
  - Output patterns (technical + mpp_report)
  - Security guardrails (encryption, IAM least privilege, no hardcoded secrets)
  - Environment-based logic rules for prod vs dev
  
- **Developer Prompt**: LLM agent instructions for code generation, validation, and refactoring
  
- **User Prompt Templates**: 3 variants (primary, strict, flexible) for different use cases
  
- **Context Injection Strategy**: RAG patterns for integrating existing module code
  
- **Guardrails**: Specific refusal patterns for PII, secrets, overly permissive IAM, network security
  
- **Few-Shot Examples**: 
  - Example 1: RDS module generation with environment logic
  - Example 2: EKS validation with multi-module consistency checks
  
- **Negative Examples**: 2 common mistakes (hardcoded passwords, missing dependencies) with corrections
  
- **Evaluation Set** (`eval_set.jsonl`): 10 test cases covering:
  - RDS PostgreSQL (dev, prod scenarios)
  - EKS cluster (HA, OIDC, private endpoints)
  - Bastion host (SSH, IRSA, CloudWatch)
  - Init module (VPC, KMS, VPC endpoints)
  - Bridge modules (PostgreSQL+pgvector, Weaviate Helm)
  - Validation tests (security review, cost optimization, cross-module consistency)
  
- **Evaluation Runner** (`eval_runner.md`): Scoring rubric and procedures for:
  - Terraform syntax validation (25pts)
  - Security & compliance review (25pts)
  - Module dependency correctness (20pts)
  - Environment-based logic (15pts)
  - Documentation quality (15pts)
  - Version comparison methodology
  - Acceptance thresholds (80+ = mergeable, 70-79 = needs review, <70 = reject)
  
- **README.md**: Complete guide covering:
  - Purpose and usage (code generation, validation, refactoring)
  - Semantic versioning rules (MAJOR/MINOR/PATCH)
  - How to bump versions and tag releases
  - Local evaluation procedures (manual and automated)
  - Rollback instructions
  - 5 common failure modes with fixes and workarounds
  - FAQ for multi-cloud, CI/CD integration, etc.

### Design Decisions
- **Temperature 0.2**: Ensures deterministic, production-safe code; minimizes hallucinations
- **Multi-model support**: Designed to work with Claude, GPT-4, and open-source models (with caveats)
- **Organizational specificity**: Tailored to your repo's patterns (init→bastion→eks→rds→weaviate dependency chain)
- **Security-first guardrails**: Non-negotiable constraints (encryption, least-privilege IAM, no secrets)
- **Cost-aware**: Environment-based sizing decisions built into system prompt

### Known Limitations
- Azure/GCP modules not yet supported (AWS-only in v1.0.0)
- Limited guidance on stateful services beyond RDS/Kubernetes
- CI/CD automation (terraform validate in pipeline) documented but not included as runnable code

---

## [Unreleased]

### Planned for v1.1.0
- [ ] Cross-module validation function (detect missing upstream outputs automatically)
- [ ] Cost comparison tool (calculate savings: prod vs dev sizing)
- [ ] Helm provider integration guide (for Weaviate, postgres-operator)
- [ ] Tighter tfsec integration (validate tfsec rules programmatically)
- [ ] Multi-region support guidance

### Planned for v2.0.0 (Breaking Changes)
- [ ] Support for multi-cloud (Azure, GCP) with shared base standards
- [ ] New variable pattern supporting optional providers (consul, vault)
- [ ] Stricter output validation (enforce exact schema matching)

---

## Version History

| Version | Date | Key Changes |
|---------|------|-------------|
| 1.0.0 | 2026-03-13 | Initial release; AWS Terraform + org patterns |

---

## How to Contribute Changes

1. **Identify the change type**:
   - MAJOR: Breaking change to module input patterns or mandatory constraints
   - MINOR: New optional patterns, updated best practices, or new evaluation categories
   - PATCH: Bug fixes, clarifications, typo corrections

2. **Make changes**:
   - Update relevant sections in `prompt_pack.yaml`
   - Add test cases to `eval_set.jsonl`
   - Update `README.md` with any new procedures
   - Update this `CHANGELOG.md`

3. **Test thoroughly**:
   - Run at least 5 evaluation test cases with new prompt version
   - Compare scores against baseline (v1.0.0)
   - Ensure no regression (new version should not decrease scores)

4. **Commit and tag**:
   ```bash
   git commit -m "feat: add Helm provider guidance (v1.1.0-rc1)"
   git tag prompts/terraform-module-generation/v1.1.0-rc1
   git push origin main --tags
   ```

5. **Communication**:
   - Post summary in team Slack/email
   - Highlight breaking changes (if MAJOR)
   - Provide migration guide for team

---

## Support & Issues

If a prompt version generates broken code or bad patterns:

1. **File an issue** (in your repo's issue tracker):
   ```
   Title: "Prompt pack v1.0.0: EKS module missing OIDC provider"
   Severity: High
   Test case: eval_set.jsonl#eks-ha-prod-001
   Reproduction: [user prompt that triggers issue]
   ```

2. **Quick workaround**: Use stable version tag
   ```bash
   git checkout prompts/terraform-module-generation/v1.0.0
   ```

3. **Fix & release**: Create PR with prompt improvements, test against eval set, release as PATCH or MINOR

---

End of CHANGELOG
