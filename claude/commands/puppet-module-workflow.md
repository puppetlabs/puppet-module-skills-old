Develop or modify code in this Puppet module following the standard workflow.

Steps:
1. Read the module structure to understand the codebase (manifests/, lib/, spec/)
2. Make the requested changes
3. Run local validation:
   - `bundle exec rake syntax lint metadata_lint check:symlinks check:git_ignore check:dot_underscore check:test_file rubocop`
   - `bundle exec rake parallel_spec`
4. Fix any lint/rubocop/unit test failures
5. If acceptance tests are relevant and a target VM is available in spec/fixtures/litmus_inventory.yaml:
   - Run `bundle exec rake litmus:acceptance:parallel`
6. Report results

Coding conventions:
- Puppet: Optional types with undef default, guard with `if $param != undef`, align => arrows
- Ruby/RSpec: frozen_string_literal, let(:var) not @var, spec_helper_acceptance for acceptance
- PowerShell: Always -EncodedCommand, $ErrorActionPreference='Stop', never SilentlyContinue
- Idempotency: `expect { idempotent_apply(manifest) }.not_to raise_error`
- Commit: `[TICKET-ID] Brief description`, never commit litmus_inventory.yaml

$ARGUMENTS
