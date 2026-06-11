# ADR 002: Ansible Testing Strategy

## Status
Accepted

## Context
We need to validate Ansible playbooks automatically and ensure they are idempotent.
Playbooks manage EC2 instances and must be tested before CI deployment.

## Decision
We use Molecule for role testing and ansible-lint for code quality:

### ansible-lint
- Runs on all playbooks in CI: `ansible-lint --recursive`
- Enforces idempotency with `creates`/`removes` directives
- Validates YAML structure and naming conventions

### Molecule (future)
- `molecule/default/converge.yml` for testing roles
- `molecule/default/verify.yml` for validation assertions
- Docker driver for fast local testing

### Playbook Idempotency
```yaml
- name: Create EC2 instances
  ec2_instance:
    ...
  creates: "/tmp/ec2_created"  # Prevents re-running if already done
  when: ec2_state == 'present'
```

## Consequences
- Playbooks are tested in ephemeral environments
- Code quality is enforced in CI before deployment
- Idempotent playbooks can be run multiple times safely
- Check mode (`--check`) available for dry-run validation

## Implementation
- `.ansible-lint` config file at repo root
- `molecule/default/molecule.yml` scenario configuration
- GitHub Actions workflow runs ansible-lint on push/PR

## References
- ansible-lint rules: https://ansible-lint.readthedocs.io/
- Molecule documentation: https://molecule.readthedocs.io/
- Ansible idempotency patterns