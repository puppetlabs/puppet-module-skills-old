---
name: puppet-module-workflow
description: 'Development workflow for Puppet modules (PDK-based). Use when: writing or editing manifests, types, providers, functions, facts, tasks, plans; running lint, unit, or acceptance tests; fixing CI failures or PR review comments; adding parameters or resources; using Litmus for remote testing; creating or updating Puppet module code.'
argument-hint: 'Describe what you want to do, e.g. "fix failing unit test" or "add new parameter to init.pp"'
---

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
examples/               # Usage examples
spec/                   # Tests
  spec/unit/            # Unit tests (rspec-puppet)
  spec/acceptance/      # Acceptance tests (Litmus)
  spec/classes/         # Class spec tests
  spec/defines/         # Defined type spec tests
  spec/functions/       # Function spec tests
  spec/fixtures/        # Test fixtures + litmus_inventory.yaml
metadata.json           # Module metadata (name, version, deps, OS support)
hiera.yaml              # Module-level hiera config
Gemfile                 # Ruby dependencies
Rakefile                # Rake task definitions
.rubocop.yml            # RuboCop configuration
.puppet-lint.rc         # Puppet-lint configuration
```

## Workflow Steps

### 1. Before Any Change

```bash
cd <module-root>
bundle install                            # Ensure deps are current
git checkout -b <TICKET-ID>              # Branch from main/default with ticket reference
```

### 2. Make Changes

#### Manifest Changes

- Parameters: use `Optional` types, default to `undef`
- Guard resources with `if $param != undef { ... }`
- Align `=>` arrows consistently within resource blocks
- Follow existing code style in the module

#### Type/Provider Changes

- Types in `lib/puppet/type/<name>.rb`
- Providers in `lib/puppet/provider/<type>/<provider>.rb`
- Add corresponding unit tests in `spec/unit/puppet/type/` and `spec/unit/puppet/provider/`

#### Template Changes

- ERB templates: `templates/<name>.erb`
- EPP templates: `templates/<name>.epp`
- Test template rendering in unit tests

#### Function Changes

- Puppet 4.x API functions: `lib/puppet/functions/<name>.rb`
- Test in `spec/functions/<name>_spec.rb`

### 3. Local Validation (No VM Required)

Always run before committing or remote testing:

```bash
# Full lint + style suite (must all pass — matches CI)
bundle exec rake syntax lint metadata_lint check:symlinks check:git_ignore check:dot_underscore check:test_file rubocop

# Unit tests
bundle exec rake parallel_spec
```

**Fix common issues:**
- RuboCop offenses: `bundle exec rubocop -a <file>` (auto-correct)
- Puppet arrow alignment: align `=>` to consistent column within resource blocks
- `RSpec/InstanceVariable`: use `let(:var) { ... }` instead of `@var` in specs

### 4. Acceptance Testing (Litmus)

**Requires:** A provisioned target VM. User maintains `spec/fixtures/litmus_inventory.yaml`.

**Run in this order for efficient feedback:**

```bash
# Fast: run only the spec you changed
TARGET_HOST='<host>' bundle exec rspec spec/acceptance/<specific_spec>.rb

# Verify connectivity
bundle exec rake "litmus:check_connectivity[<host>]"

# Full suite (final gate before push)
bundle exec rake litmus:acceptance:parallel
```

**If tests fail:**
1. Read the error message — Litmus surfaces stdout/stderr from the target
2. Fix the code
3. Re-run only the failing spec (faster), then full suite once passing

**If target is unreachable:** VM may have expired. Ask user to re-provision and update `litmus_inventory.yaml`.

### 5. Commit and Push

```bash
# Stage relevant files — NEVER commit litmus_inventory.yaml or local-only config
git add <changed files>

# Commit with ticket reference
git commit -m "[TICKET-ID] Brief description of change"

# Push
git push -u origin <branch>
```

### 6. PR and Review

1. Create PR targeting the default branch
2. Wait for CI + Copilot auto-review
3. For valid review comments: fix, re-validate (steps 3+4), push
4. For invalid comments: reply with reasoning
5. Run full validation again before final push

## Unit Test Patterns

### Class Tests (`spec/classes/`)

```ruby
# frozen_string_literal: true

require 'spec_helper'

describe 'module_name' do
  on_supported_os.each do |os, os_facts|
    context "on #{os}" do
      let(:facts) { os_facts }
      let(:params) { { param_name: 'value' } }

      it { is_expected.to compile.with_all_deps }
      it { is_expected.to contain_resource_type('title').with(key: 'value') }
    end
  end
end
```

### Defined Type Tests (`spec/defines/`)

```ruby
describe 'module_name::defined_type' do
  let(:title) { 'resource_title' }
  let(:params) { { param: 'value' } }

  it { is_expected.to compile.with_all_deps }
end
```

## Acceptance Test Patterns

### Idempotency

Always use: `expect { idempotent_apply(manifest) }.not_to raise_error`

### Verifying Applied State

After `apply_manifest`, use `run_shell` to check actual system state:

```ruby
result = run_shell('command_to_check_state')
expect(result.stdout.strip).to eq('expected_value')
```

### PowerShell on Windows Targets

Always use `-EncodedCommand` to avoid shell interpolation issues:

```ruby
ps_script = <<~POWERSHELL
  $ErrorActionPreference = 'Stop'
  try {
    # PowerShell code with $variables
  } catch {
    Write-Error $_
    exit 1
  }
POWERSHELL
encoded = Base64.strict_encode64(ps_script.encode('UTF-16LE'))
result = run_shell("powershell -NoProfile -EncodedCommand #{encoded}", expect_failures: true)
```

Never use `-ErrorAction SilentlyContinue` — it hides errors.

### Cleanup Pattern

Use `begin/ensure` for any temporary resources:

```ruby
temp_resource = create_temp_resource
begin
  # assertions using temp_resource
ensure
  cleanup_temp_resource(temp_resource) if temp_resource
end
```

## Coding Conventions

### Ruby / RSpec

- `frozen_string_literal: true` at top of every spec file
- `let(:var) { ... }` not instance variables (`@var`)
- Acceptance specs: `require 'spec_helper_acceptance'`
- Unit specs: `require 'spec_helper'`

### Puppet

- Parameters: `Optional[Type]` with `undef` default
- Guard resources: `if $param != undef { ... }`
- String quoting: single quotes unless interpolation needed
- Module dependencies declared in `metadata.json`

### Metadata

- Keep `metadata.json` version, OS support, and dependency versions current
- `operatingsystem_support` must match what tests actually cover

## Common Pitfalls

| Problem | Cause | Solution |
|---------|-------|----------|
| PowerShell `$` variables stripped | Shell interpolation | Use `-EncodedCommand` |
| RuboCop `InstanceVariable` | `@var` in RSpec examples | Use `let(:var)` |
| Puppet lint arrow alignment | Inconsistent `=>` column | Align within same resource block |
| Litmus auth failure | VM expired or credentials stale | Re-provision, update inventory |
| `metadata.json` lint fail | Missing field or bad version | Check with `metadata_lint` |
| Catalog compile error | Missing dependency module | Add to `.fixtures.yml` and `metadata.json` |
| Acceptance test not idempotent | Resource triggers change every run | Check resource parameters for defaults |

## CI Commands Reference

```bash
# Full lint suite (matches CI)
bundle exec rake syntax lint metadata_lint check:symlinks check:git_ignore check:dot_underscore check:test_file rubocop

# Unit tests
bundle exec rake parallel_spec

# Acceptance (local Litmus)
TARGET_HOST='<host>' bundle exec rspec spec/acceptance/<file>
bundle exec rake litmus:acceptance:parallel

# List all available rake tasks
bundle exec rake --tasks
```
