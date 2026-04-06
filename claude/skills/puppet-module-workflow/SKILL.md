# Puppet Module Development Workflow

## When to Use

Any change to a Puppet module: manifests, types, providers, functions, facts, tasks, plans, tests, docs, metadata, or CI config.

## Standard Module Structure

```
manifests/              # Puppet classes and defined types
lib/puppet/type/        # Custom resource types
lib/puppet/provider/    # Resource providers
lib/puppet/functions/   # Custom Puppet functions
lib/facter/             # Custom facts
tasks/                  # Bolt tasks
plans/                  # Bolt plans
templates/              # ERB/EPP templates
files/                  # Static files
types/                  # Puppet type aliases
data/                   # Hiera data (module-level)
spec/                   # Tests
  spec/unit/            # Unit tests (rspec-puppet)
  spec/acceptance/      # Acceptance tests (Litmus)
  spec/classes/         # Class spec tests
  spec/defines/         # Defined type spec tests
  spec/functions/       # Function spec tests
  spec/fixtures/        # Test fixtures + litmus_inventory.yaml
metadata.json           # Module metadata
Gemfile                 # Ruby dependencies
Rakefile                # Rake task definitions
```

## Workflow Steps

### 1. Before Any Change

```bash
cd <module-root>
bundle install
git checkout -b <TICKET-ID>
```

### 2. Local Validation (No VM Required)

Always run before committing or remote testing:

```bash
# Full lint + style suite (must all pass — matches CI)
bundle exec rake syntax lint metadata_lint check:symlinks check:git_ignore check:dot_underscore check:test_file rubocop

# Unit tests
bundle exec rake parallel_spec
```

Auto-fix style issues: `bundle exec rubocop -a <file>`

### 3. Acceptance Testing (Litmus)

Requires a provisioned target VM. Inventory at `spec/fixtures/litmus_inventory.yaml`.

```bash
# Fast: single spec
TARGET_HOST='<host>' bundle exec rspec spec/acceptance/<specific_spec>.rb

# Full suite
bundle exec rake litmus:acceptance:parallel
```

If tests fail: read error, fix, re-run single spec first, then full suite.

### 4. Commit and Push

```bash
git add <changed files>    # NEVER commit litmus_inventory.yaml
git commit -m "[TICKET-ID] Brief description"
git push -u origin <branch>
```

## Coding Conventions

### Puppet
- Parameters: `Optional[Type]` with `undef` default
- Guard resources: `if $param != undef { ... }`
- Align `=>` arrows within resource blocks
- Single quotes unless interpolation needed

### Ruby / RSpec
- `frozen_string_literal: true` at top of every spec
- `let(:var) { ... }` not `@var` instance variables
- Acceptance specs: `require 'spec_helper_acceptance'`
- Unit specs: `require 'spec_helper'`
- Idempotency: `expect { idempotent_apply(manifest) }.not_to raise_error`

### PowerShell on Windows Targets
- Always use `-EncodedCommand` (avoids shell `$` interpolation)
- Use `$ErrorActionPreference = 'Stop'` with `try/catch`
- Never use `-ErrorAction SilentlyContinue`

## Common Pitfalls

| Problem | Solution |
|---------|----------|
| PowerShell `$` variables stripped by shell | Use `-EncodedCommand` |
| RuboCop `InstanceVariable` offense | Use `let(:var)` in RSpec |
| Puppet lint arrow alignment | Align `=>` within same resource block |
| Litmus VM auth failure | Re-provision, update inventory |
| Catalog compile error from missing dep | Add to `.fixtures.yml` and `metadata.json` |
| Acceptance test not idempotent | Check resource params for unintended defaults |

## CI Commands

```bash
bundle exec rake syntax lint metadata_lint check:symlinks check:git_ignore check:dot_underscore check:test_file rubocop
bundle exec rake parallel_spec
bundle exec rake litmus:acceptance:parallel
```
