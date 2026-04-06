---
name: security-policy-workflow
description: 'Development workflow for the puppetlabs-security_policy Puppet module. Use when: making any code change (manifests, types, providers, data, tests, docs); running lint, unit, or acceptance tests; fixing CI or PR review issues; adding parameters or resources; working with Windows registry or secedit settings; using Litmus for remote testing.'
argument-hint: 'Describe what you want to do, e.g. "fix failing unit test" or "add new lockout parameter"'
---

# puppetlabs-security_policy — Development Workflow

## When to Use

Any change to this module: code, tests, docs, metadata, CI config, or dependency updates.

## Module Structure

```
manifests/init.pp                           # Main class — all parameters and resources
lib/puppet/type/securityoption.rb           # Custom type (secedit-managed settings)
lib/puppet/provider/securityoption/         # Provider implementation
lib/puppet_x/security_policy/data.rb        # Data store: setting name → INF section/key
spec/files/SecurityPolicy.inf               # Reference INF (all setting names/sections)
spec/unit/                                  # Unit tests (type, provider)
spec/acceptance/                            # Acceptance tests (Litmus, Windows targets)
spec/spec_helper_acceptance_local.rb        # Acceptance helper functions
spec/fixtures/litmus_inventory.yaml         # Litmus target inventory (user-managed, never commit)
examples/comprehensive_security_policy.pp   # Full parameter reference example
```

### Resource Types

| Resource | Backing Store | How Values Are Set | How to Verify |
|----------|--------------|-------------------|---------------|
| `registry::value` | Windows Registry (HKLM) | Puppet `registry::value` defined type | PowerShell `Get-ItemProperty` |
| `securityoption` | Local Security Policy DB | `secedit /configure` with generated INF | `secedit /export` + INF parse |

## Workflow Steps

### 1. Before Any Change

```bash
cd <repo-root>
bundle install                            # Ensure deps are current
git checkout -b MODULES-XXXXX             # Branch from main with JIRA ticket
```

### 2. Make Changes

Depending on what you're changing, follow the relevant section below.

#### Manifest Changes (`manifests/init.pp`)

- Every parameter must have an `Optional` type and default to `undef`
- Guard each resource with `if $param != undef { ... }`
- Align `=>` arrows consistently within each resource block
- Registry resources use `registry::value` (from `puppetlabs-registry`)
- System Access resources use `securityoption` (custom type in this module)

#### Type/Provider Changes (`lib/`)

- Type: `lib/puppet/type/securityoption.rb`
- Provider: `lib/puppet/provider/securityoption/securityoption.rb`
- Data store: `lib/puppet_x/security_policy/data.rb` — maps setting names to INF sections and keys
- Unit tests: `spec/unit/puppet/type/` and `spec/unit/puppet/provider/`

#### Test Changes (`spec/`)

- Unit tests use `rspec-puppet` with facts from `spec/default_facts.yml`
- Acceptance tests use Puppet Litmus against live Windows VMs
- All acceptance helpers live in `spec/spec_helper_acceptance_local.rb`

### 3. Local Validation (No VM Required)

Always run these before any remote testing or committing:

```bash
# Lint + style (must match CI exactly)
bundle exec rake syntax lint metadata_lint check:symlinks check:git_ignore check:dot_underscore check:test_file rubocop

# Unit tests
bundle exec rake parallel_spec
```

To auto-fix style issues: `bundle exec rubocop -a <file>`

### 4. Acceptance Testing (Litmus)

**Requires:** A provisioned Windows Server 2019/2022 VM with SSH access. User maintains `spec/fixtures/litmus_inventory.yaml` with target details.

**Run in this order for efficient feedback:**

```bash
# Fast: run only the spec you changed
TARGET_HOST='<ip>:22' bundle exec rspec spec/acceptance/<specific_spec>.rb

# Verify connectivity
bundle exec rake "litmus:check_connectivity[<ip>:22]"

# Full suite (final gate)
bundle exec rake litmus:acceptance:parallel
```

**If tests fail:** Read the error (helpers surface stderr/stdout), fix, and re-run the single failing spec before retrying the full suite.

**If VM is unreachable:** The provisioned VM may have expired. Ask the user to re-provision and update `litmus_inventory.yaml`.

### 5. Commit and Push

```bash
# Stage relevant files — NEVER commit litmus_inventory.yaml
git add <changed files>
git commit -m "[MODULES-XXXXX] Brief description"
git push -u origin MODULES-XXXXX
```

### 6. PR and Review

1. Create PR targeting `main`
2. Copilot will auto-review — evaluate each comment for validity
3. For valid comments: fix, re-validate (step 3+4), push
4. For invalid comments: reply with reasoning

## Acceptance Test Patterns

### Helpers (`spec/spec_helper_acceptance_local.rb`)

| Function | Purpose |
|----------|---------|
| `get_registry_value(path, value_name)` | Query HKLM registry; raises with stderr on failure |
| `export_security_policy(test_id)` | Run `secedit /export`; returns temp INF path; raises on failure |
| `read_security_policy_value(name, path)` | Parse one setting from an exported INF |
| `cleanup_security_policy_export(path)` | Delete temp INF; always call in `ensure` block |

### Registry Value Assertions

```ruby
registry_expectations = [
  ['HKLM:\\Path\\To\\Key', 'ValueName', 'expected_value'],
]
registry_expectations.each do |path, value_name, expected|
  actual = get_registry_value(path, value_name)
  expect(actual).to eq(expected), "Expected #{path} #{value_name}=#{expected}, got #{actual.inspect}"
end
```

For array/multi-value registry keys, use `expect(actual).to include('substring')`.

### System Access (secedit) Assertions

Export once, read many, clean up with `ensure`:

```ruby
temp_path = export_security_policy(test_id)
begin
  [['MinimumPasswordAge', '1'], ['PasswordComplexity', '1']].each do |name, expected|
    actual = read_security_policy_value(name, temp_path)
    expect(actual).to eq(expected), "Expected #{name}=#{expected}, got #{actual.inspect}"
  end
ensure
  cleanup_security_policy_export(temp_path) if temp_path
end
```

### Idempotency

Always use: `expect { idempotent_apply(manifest) }.not_to raise_error`

### Test File Organization

| File | What It Tests |
|------|---------------|
| `security_policy_idempotency_spec.rb` | Manifest applies cleanly, second run produces no changes |
| `security_policy_registry_settings_spec.rb` | Registry values match expected after apply |
| `security_policy_system_settings_spec.rb` | secedit-exported values match expected after apply |

## Coding Conventions

### PowerShell in Ruby Helpers

Always use `-EncodedCommand` to avoid shell interpolation issues:

```ruby
ps_script = <<~POWERSHELL
  $ErrorActionPreference = 'Stop'
  try {
    # PowerShell code here
  } catch {
    Write-Error $_
    exit 1
  }
POWERSHELL
encoded = Base64.strict_encode64(ps_script.encode('UTF-16LE'))
result = run_shell("powershell -NoProfile -EncodedCommand #{encoded}", expect_failures: true)
```

Never use `-ErrorAction SilentlyContinue` — it hides errors that Ruby code needs to detect.

### RSpec Style

- Use `let(:var) { ... }` not instance variables (`@var`)
- Use `frozen_string_literal: true` at top of every spec file
- Acceptance specs require `spec_helper_acceptance`, unit specs require `spec_helper`

### Puppet Style

- All parameters default to `undef` with `Optional` types
- Guard resources with `if $param != undef`
- Align `=>` arrows within resource blocks
- Disabled lint checks: `relative`, `80chars`, `140chars`, `documentation`, `autoloader_layout`

## Common Pitfalls

| Problem | Cause | Solution |
|---------|-------|----------|
| `Continue=Stop` parse error in PS | Shell strips `$` variables | Use `-EncodedCommand` |
| `Select-String` returns no match | `-SimpleMatch` treats `^` as literal | Use regex mode (no `-SimpleMatch`) |
| Litmus VM auth failure | VM expired or wrong credentials | Re-provision, update `litmus_inventory.yaml` |
| RuboCop `InstanceVariable` offense | `@var` in RSpec examples | Use `let(:var)` |
| Registry helper returns `nil` | Backslashes not escaped | `.gsub('\\', '\\\\')` in path |
| secedit setting not found | Case mismatch in setting name | Cross-check `spec/files/SecurityPolicy.inf` |
| Puppet lint arrow alignment | Inconsistent `=>` column | Align all arrows in same resource block |

## CI Reference

These commands match what GitHub Actions CI runs:

```bash
# Lint + style
bundle exec rake syntax lint metadata_lint check:symlinks check:git_ignore check:dot_underscore check:test_file rubocop

# Unit tests
bundle exec rake parallel_spec

# Acceptance (local)
TARGET_HOST='<ip>:22' bundle exec rspec spec/acceptance/<file>
bundle exec rake litmus:acceptance:parallel
```
