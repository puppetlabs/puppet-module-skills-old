# puppet-module-build-skills

Reusable AI coding assistant skills for Puppet module development. Provides workflow automation for both **VS Code Copilot Chat** and **Claude Code**.

## Repository Structure

```
puppet-module-build-skills/
├── README.md
├── copilot/
│   └── skills/
│       ├── puppet-module-workflow/        # Generic — works with any Puppet module
│       │   └── SKILL.md
│       └── security-policy-workflow/      # Module-specific example
│           └── SKILL.md
└── claude/
    ├── skills/
    │   └── puppet-module-workflow/        # Generic skill (CLAUDE.md-style reference)
    │       └── SKILL.md
    └── commands/
        └── puppet-module-workflow.md      # Slash command for Claude Code
```

## Setup

### VS Code Copilot Chat

#### Option A: Personal skill (available in all workspaces)

Symlink the generic skill into your Copilot personal skills folder:

```bash
# Create the personal skills directory if it doesn't exist
mkdir -p ~/.copilot/skills

# Symlink the generic skill
ln -s /path/to/puppet-module-build-skills/copilot/skills/puppet-module-workflow \
      ~/.copilot/skills/puppet-module-workflow
```

After symlinking, reload VS Code (`Cmd+Shift+P` → "Developer: Reload Window"). You can then type `/puppet-module-workflow` in Copilot Chat.

#### Option B: Per-module skill (committed to a specific module repo)

Copy the module-specific skill into the module's `.github/skills/` directory:

```bash
cd /path/to/puppetlabs-<module>

# Copy the generic skill
cp -r /path/to/puppet-module-build-skills/copilot/skills/puppet-module-workflow \
      .github/skills/puppet-module-workflow

# Or copy a module-specific skill
cp -r /path/to/puppet-module-build-skills/copilot/skills/security-policy-workflow \
      .github/skills/security-policy-workflow
```

> **Note:** VS Code discovers skills at `.github/skills/` relative to **workspace folder roots**. If your workspace root is a parent directory containing multiple modules, put skills in the parent's `.github/skills/` or add each module as a separate workspace folder.

### Claude Code

#### Option A: Personal command (available in all projects)

Symlink the command into your Claude personal config:

```bash
mkdir -p ~/.claude/commands

ln -s /path/to/puppet-module-build-skills/claude/commands/puppet-module-workflow.md \
      ~/.claude/commands/puppet-module-workflow.md
```

Then use `/puppet-module-workflow` in Claude Code.

#### Option B: Per-project command

Copy into a specific module's `.claude/commands/` directory:

```bash
cd /path/to/puppetlabs-<module>
mkdir -p .claude/commands

cp /path/to/puppet-module-build-skills/claude/commands/puppet-module-workflow.md \
   .claude/commands/puppet-module-workflow.md
```

#### Option C: Project instructions (always-on)

Copy the skill content into a module's `CLAUDE.md` for always-on instructions:

```bash
cp /path/to/puppet-module-build-skills/claude/skills/puppet-module-workflow/SKILL.md \
   /path/to/puppetlabs-<module>/CLAUDE.md
```

## Organizing Module-Specific Skills

The recommended approach for module-specific skills:

### 1. Add to this repo

Create a new folder under `copilot/skills/` and `claude/commands/`:

```
copilot/skills/<module-name>-workflow/SKILL.md
claude/commands/<module-name>-workflow.md
```

The generic skill covers standard Puppet module patterns. Module-specific skills add:
- Module-specific resource types and their verification methods
- Custom acceptance test helpers
- Module-specific coding patterns or conventions
- Integration details (e.g., Windows secedit, PostgreSQL commands, Apache configs)

### 2. Naming convention

```
<module-short-name>-workflow    # e.g., security-policy-workflow, apache-workflow
```

### 3. Module-specific skill template

When creating a new module-specific skill, include:

```markdown
---
name: <module>-workflow
description: 'Development workflow for puppetlabs-<module>. Use when: <module-specific triggers>.'
argument-hint: 'Describe what you want to do'
---

# puppetlabs-<module> — Development Workflow

## Module-Specific Architecture
<!-- Resource types, backing stores, verification methods -->

## Module-Specific Patterns
<!-- Custom helpers, assertion patterns, etc. -->

## Module-Specific Pitfalls
<!-- Known issues unique to this module -->
```

For general workflow steps (lint, test, commit, PR), reference the generic `puppet-module-workflow` skill rather than duplicating.

### 4. Example modules with specific skills

| Module | Skill Name | Key Additions |
|--------|-----------|---------------|
| `security_policy` | `security-policy-workflow` | Windows secedit, registry helpers, EncodedCommand patterns |
| `apache` | `apache-workflow` | Vhost testing, SSL config, cross-OS template differences |
| `postgresql` | `postgresql-workflow` | Database setup/teardown, SQL verification queries |

## What Each Skill Covers

### Generic (`puppet-module-workflow`)
- Standard PDK module structure
- Lint, syntax, rubocop, metadata_lint validation
- Unit test patterns (rspec-puppet: classes, defines, functions)
- Acceptance test patterns (Litmus)
- Git workflow and PR conventions
- Common pitfalls and fixes

### Module-Specific (e.g., `security-policy-workflow`)
- Module architecture and resource types
- Custom acceptance test helpers
- Verification methods (registry queries, secedit export, etc.)
- Module-specific coding conventions
- Module-specific pitfalls

## Updating Skills

After modifying skills in this repo:

1. If using **symlinks** — changes take effect after VS Code reload
2. If using **copies** — re-copy the updated files to each module

## Contributing

1. Fork and branch
2. Add or update skills under `copilot/skills/` and `claude/commands/`
3. Test by symlinking into your personal skills folder
4. Open a PR
