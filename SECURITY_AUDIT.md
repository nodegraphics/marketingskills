# Security Audit Report: Marketing Skills (Claude Code Plugin)

**Prepared for**: Security Review Team
**Prepared by**: Emrah Gonulkirmaz
**Date**: 2025-03-05
**Repository**: https://github.com/nodegraphics/marketingskills
**Upstream**: https://github.com/coreyhaines31/marketingskills
**License**: MIT
**Version**: 1.0.0-hardened

---

## 1. Executive Summary

This report documents the security audit and hardening of an open-source Claude Code skills plugin ("Marketing Skills" by Corey Haines) before deployment in our company environment. The plugin provides 32 marketing advisory skills (copywriting, SEO, CRO, advertising, etc.) and 51 CLI integration tools for third-party marketing services.

**Verdict**: The upstream repository contains no malicious code, prompt injection, data exfiltration, or backdoors. Three medium/low-severity issues were identified and remediated in our fork before deployment.

**Audit scope**: 294 files (32 skill definitions, 51 JS CLI tools, 66 integration guides, GitHub Actions workflows, shell scripts, plugin configuration, all markdown instruction files).

---

## 2. What This Plugin Does

When installed as a Claude Code skill, this plugin:

- **Loads 32 SKILL.md files** as context when Claude detects relevant marketing topics (e.g., user says "optimize this landing page" and the `page-cro` skill activates)
- **Each skill is pure markdown** containing frameworks, templates, and structured guidance for marketing tasks
- **No code executes automatically** during normal skill use. Skills are read-only text that guide Claude's responses
- **51 optional CLI tools** (Node.js scripts in `tools/clis/`) can be invoked by Claude to call third-party marketing APIs (GA4, Stripe, Mailchimp, etc.) but only when corresponding API keys are set as environment variables

---

## 3. Architecture and Data Flow

```
User prompt
    |
    v
Claude Code detects marketing topic
    |
    v
Loads matching SKILL.md (read-only markdown, no execution)
    |
    v
Claude generates marketing advice using skill as context
    |
    v
[Optional] User asks Claude to run a CLI tool
    |
    v
tools/clis/<service>.js reads API key from env var, calls external API
```

**Key properties**:
- Skills are passive text, not executable code
- No code runs on plugin install or load
- No hooks (pre-tool, post-tool) are registered
- No MCP servers are bundled or configured
- CLI tools are opt-in (dormant without env vars)

---

## 4. Audit Methodology

Three parallel security audits were conducted:

| Audit Pass | Scope | Files Examined |
|------------|-------|----------------|
| **Code audit** | All 51 JS CLI tools + GitHub Actions scripts | 63 JS files |
| **Prompt audit** | CLAUDE.md, AGENTS.md, all 32 SKILL.md files | 34 markdown files |
| **Infrastructure audit** | Plugin config, shell scripts, GitHub workflows, .gitignore | 12 config/infra files |

Each pass checked for: prompt injection, data exfiltration, command injection, credential theft, obfuscated code, backdoors, hidden MCP servers, malicious hooks, auto-execution, excessive permissions, social engineering, and scope creep.

---

## 5. Findings (Pre-Hardening)

### Finding 1: Remote Phone-Home on Every Session (MEDIUM, REMEDIATED)

**Location**: `AGENTS.md` lines 192-214 (loaded as `CLAUDE.md` via symlink)

**Issue**: The upstream instructed Claude to fetch `https://raw.githubusercontent.com/coreyhaines31/marketingskills/main/VERSIONS.md` once per session on first skill use, and to run `git pull` if the user said "update skills."

**Risks**:
- IP address leaked to GitHub on every session
- Upstream repo compromise could push malicious content via `git pull`
- Outbound network request initiated without explicit user consent

**Remediation**: Removed the entire auto-update section. Replaced with manual-only update instructions. Users must explicitly check and pull updates on their own schedule.

**Commit**: `981af45` ("sec: harden fork for enterprise use")

---

### Finding 2: Broad Codebase Scanning (MEDIUM, REMEDIATED)

**Location**: `skills/product-marketing-context/SKILL.md` line 36

**Issue**: The `product-marketing-context` skill instructed Claude to "Read the codebase: README, landing pages, marketing copy, about pages, meta descriptions, package.json, any existing docs" with no scope restriction. All 31 other skills then read the generated context file on every invocation.

**Risks**:
- Claude could read files containing confidential data (internal docs, strategy files, config with credentials)
- Generated context file (`.agents/product-marketing-context.md`) persists on disk and could be accidentally committed

**Remediation**:
1. Scoped the reading instruction to "ONLY marketing-relevant files" with explicit exclusion of source code, environment files, credential configs, and internal strategy docs
2. Added `.agents/` to `.gitignore` to prevent accidental commits of generated context

**Commit**: `981af45` ("sec: harden fork for enterprise use")

---

### Finding 3: CLI Tools Use API Keys from Environment (LOW, ACCEPTED)

**Location**: All 51 files in `tools/clis/`

**Issue**: Each CLI tool reads API keys from environment variables (e.g., `process.env.GA4_ACCESS_TOKEN`, `process.env.STRIPE_API_KEY`) and makes HTTP calls to the corresponding third-party service.

**Risk**: If environment variables are set, Claude can invoke these tools to make live API calls to production services.

**Assessment**: This is standard practice and by design. Tools are dormant without env vars. Each tool only contacts its documented service (verified: no phone-home to plugin author). No remediation needed.

**Recommendation**: Only set environment variables for services you intend to use. Do not set all 51 service keys.

---

## 6. Full Security Checklist

| Check | Result | Details |
|-------|--------|---------|
| Prompt injection | CLEAN | No "ignore instructions" or override attempts in any file |
| Data exfiltration | CLEAN | No data sent to plugin author's servers |
| Hidden unicode/zero-width characters | CLEAN | Zero matches across entire repo |
| Obfuscated code (eval, encoded payloads) | CLEAN | `base64` used only for standard HTTP Basic Auth |
| Malicious hooks in plugin config | CLEAN | No pre/post hooks registered in marketplace.json |
| Auto-execution on install/load | CLEAN | No package.json, no lifecycle scripts, no auto-run |
| Hidden MCP servers | CLEAN | No external MCP endpoints configured |
| Command injection | CLEAN | No unsanitized shell exec in any JS file |
| Credential theft | CLEAN | No reading of .env files, SSH keys, or config stores |
| GitHub Actions abuse | CLEAN | No secret exfiltration, standard CI only |
| Scope creep | CLEAN | All 32 skills strictly marketing-focused |
| File system access | CLEAN | Only `wistia.js` reads files (user-specified SRT for upload) |
| Dependencies / supply chain | CLEAN | Zero npm dependencies. All JS is self-contained |

---

## 7. Changes Made in Our Fork

| Change | File | Purpose |
|--------|------|---------|
| Removed auto-update phone-home | `AGENTS.md` | Prevents outbound network requests and unsanctioned git pull |
| Added `.agents/` to .gitignore | `.gitignore` | Prevents committing business-sensitive generated context |
| Scoped codebase scanning | `skills/product-marketing-context/SKILL.md` | Restricts file reading to marketing content only |
| Updated repository URL | `.claude-plugin/marketplace.json` | Points to our fork, versioned as `1.0.0-hardened` |
| Added upstream reference | `AGENTS.md` | Maintains attribution to original author |

**All changes preserve full functionality.** No skills, tools, or capabilities were removed.

---

## 8. Deployment Recommendations

1. **Install from our fork only**: `https://github.com/nodegraphics/marketingskills` (not the upstream)
2. **Do not set all API env vars**: Only configure environment variables for marketing services actively in use
3. **Review `.agents/` output**: If using the `product-marketing-context` skill, review the generated file before sharing it, as it summarizes business positioning
4. **Pin to commit**: For maximum stability, pin installation to commit `981af45` rather than tracking `main`
5. **Upstream sync policy**: Periodically review upstream changes (`coreyhaines31/marketingskills`) before merging into our fork. Do not auto-sync
6. **GitHub Actions**: If contributing back, pin third-party Actions to commit SHAs instead of version tags

---

## 9. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Upstream repo compromise | Low | High | Using our fork, no auto-update |
| Confidential data in generated context | Medium | Medium | Scoped scanning + .gitignore |
| Accidental API calls to production | Low | Medium | Env vars not set by default |
| Skill prompt manipulation | Very Low | Low | All skills audited, pure markdown |

**Overall risk rating**: LOW (after hardening)

---

## 10. Files Inventory

| Category | Count | Location |
|----------|-------|----------|
| Marketing skills | 32 | `skills/*/SKILL.md` |
| CLI tools (Node.js) | 51 | `tools/clis/*.js` |
| Integration guides | 66 | `tools/integrations/*.md` |
| GitHub workflows | 2 | `.github/workflows/` |
| GitHub scripts | 1 | `.github/scripts/sync-skills.js` |
| Shell scripts | 2 | `validate-skills.sh`, `validate-skills-official.sh` |
| Plugin config | 1 | `.claude-plugin/marketplace.json` |
| Documentation | 5 | `README.md`, `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `VERSIONS.md` |
| **Total** | **294** | |

---

## 11. Conclusion

The Marketing Skills plugin is a well-structured, content-only skills collection with no malicious intent or hidden behavior. The three identified issues (auto-update, broad codebase scanning, API key usage) are standard patterns in the open-source ecosystem but warranted hardening for enterprise use.

Our fork at `https://github.com/nodegraphics/marketingskills` (commit `981af45`, version `1.0.0-hardened`) addresses all medium-severity findings while preserving full functionality. The plugin is approved for deployment with the recommendations outlined in Section 8.

---

*Audit conducted using automated code analysis across 294 files with manual review of all findings.*
