# ADR 002: Ansible Testing Strategy

## Status
Accepted

## Context
We need to validate Ansible playbooks automatically and ensure they are idempotent.
Playbooks manage EC2 instances and must be tested before CI deployment.

## Decision
We use Molecule for role testing and ansible-lint for code quality:

### ansible-lint
- CI runs `ansible-lint --offline roles/ec2_provision molecule/default`
- Checks the role and Molecule scenario that ship with the repository
- Validates YAML structure and naming conventions

### Molecule (future)
- `molecule/default/converge.yml` for testing roles
- `molecule/default/verify.yml` for validation assertions
- Docker driver for fast local testing

### Playbook Idempotency
```yaml
- name: Provision EC2 instances only when lookup returned no match
  amazon.aws.ec2_instance:
    name: "{{ item.name }}"
    state: running
  when: ec2_provision_instances_found.results[loop_index].instances | length == 0

- name: Wait for SSH only for newly created instances
  ansible.builtin.wait_for:
    host: >-
      {{ item.instances[0].public_ip_address |
      default(item.instances[0].public_dns_name, true) }}
  when:
    - ec2_provision_result is changed
    - item.changed | default(false)
    - item.instances | default([]) | length > 0
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
