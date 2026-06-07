# cyberark_sia_policy_management

Ansible role for idempotently creating, updating, and deleting CyberArk Secure Infrastructure Access (SIA) access control policies via the UAP REST API (`{subdomain}.uap.cyberark.cloud/api/policies`).

## Requirements

- Ansible Core 2.9+
- A valid `cyberark_token` — use the [`cyberark_auth`](https://github.com/TobyAnscombe/cyberark-api-management) role to produce one
- `cyberark_identity_tenant` — required when resolving principals from usernames (see Principal Resolution below)

## Installation

```yaml
# requirements.yml
roles:
  - name: cyberark_auth
    src: https://github.com/TobyAnscombe/cyberark-api-management
    version: main
  - name: cyberark_sia_policy_management
    src: https://github.com/TobyAnscombe/cyberark-api-policy-management
    version: main
```

```bash
ansible-galaxy install -r requirements.yml
```

## Quick start

```yaml
- name: Manage CyberArk SIA policy
  hosts: localhost
  gather_facts: false
  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    cyberark_subdomain: "YOUR_SUBDOMAIN"

    cyberark_sia_policy_name: "john.smith - AWS EC2 SSH"
    cyberark_sia_policy_state: present

    cyberark_sia_policy_entitlement:
      targetCategory: VM
      locationType: AWS
      policyType: Recurring

    cyberark_sia_policy_principal_usernames:
      - john.smith@example.com

    cyberark_sia_policy_conditions:
      accessWindow:
        daysOfTheWeek: [1, 2, 3, 4, 5]
        fromHour: "08:00"
        toHour: "18:00"
      maxSessionDuration: 4

    cyberark_sia_policy_targets:
      AWS:
        accountIds: []
        regions: [eu-west-1]
        tags:
          - key: Environment
            value: [production]

    cyberark_sia_policy_behavior:
      connectAs:
        ssh:
          username: ec2-user

  roles:
    - cyberark_auth
    - cyberark_sia_policy_management
```

## Principal resolution

The role supports two ways to specify principals:

### Option A — full principal objects

Use when you already know the principal's CyberArk Identity details, e.g. when assigning an Identity role (type: ROLE) as a group-based principal. Role IDs can be retrieved from CyberArk Identity: Admin → Core Services → Roles.

```yaml
cyberark_sia_policy_principals:
  - id: 8a3f1b2c_4d5e_6f7a_8b9c_0d1e2f3a4b5c
    name: "SIA - Linux Server Administrators"
    sourceDirectoryName: Identity
    sourceDirectoryId: Identity
    type: ROLE
```

### Option B — username list (recommended for vaulted credentials)

Pass the username(s) as they appear on the vaulted account. The role queries the CyberArk Identity `RedRock/query` API to resolve each username to its internal `id`, `sourceDirectoryName`, and `sourceDirectoryId` automatically. Requires `cyberark_identity_tenant` to be set.

```yaml
cyberark_sia_policy_principal_usernames:
  - john.smith@example.com
```

## Variables

See [`defaults/main.yml`](defaults/main.yml) for the full reference with inline documentation. Key variables:

| Variable | Default | Description |
|---|---|---|
| `cyberark_subdomain` | `""` | Subdomain of the UAP endpoint (`htb` → `htb.uap.cyberark.cloud`) |
| `cyberark_identity_tenant` | `""` | Tenant ID for principal resolution (`aca4779`) |
| `cyberark_sia_policy_name` | `""` | Policy name — used as the unique identifier |
| `cyberark_sia_policy_state` | `present` | `present` or `absent` |
| `cyberark_sia_policy_entitlement` | see defaults | `targetCategory`, `locationType`, `policyType` |
| `cyberark_sia_policy_principals` | `[]` | Full principal objects (Option A) |
| `cyberark_sia_policy_principal_usernames` | `[]` | Usernames to resolve (Option B) |
| `cyberark_sia_policy_conditions` | see defaults | Access window and session limits |
| `cyberark_sia_policy_idle_time` | `10` | Idle timeout in minutes (VM and DB policies only) |
| `cyberark_sia_policy_targets` | `{}` | Target resources — structure varies by policy type |
| `cyberark_sia_policy_behavior` | `{}` | `connectAs` credentials for VM policies |
| `cyberark_sia_policy_access_approval` | undefined | Approval gate (requires tenant feature flag) |
| `cyberark_sia_policy_tags` | `[]` | Policy tags |
| `cyberark_sia_policy_status` | `Active` | `Active` or `Suspended` |
| `cyberark_sia_policy_validate_after` | `false` | Trigger post-create validation (Cloud Console only) |

## Supported policy types

| `targetCategory` | `locationType` | Use case |
|---|---|---|
| `VM` | `AWS` | SSH/RDP to AWS EC2 instances |
| `VM` | `Azure` | SSH/RDP to Azure VMs |
| `VM` | `GCP` | SSH/RDP to GCP Compute instances |
| `VM` | `FQDN/IP` | SSH/RDP to on-prem or non-cloud VMs |
| `DB` | `FQDN/IP` | Database access (MySQL, PostgreSQL, etc.) |
| `Cloud Console` | `AWS` | AWS IAM / IAM Identity Center access |
| `Cloud Console` | `Azure` | Azure resource role / Entra ID role access |
| `Cloud Console` | `GCP` | GCP IAM role access |
| `Groups` | `Azure` | Entra ID group membership assignment |

## Target structures

### VM — AWS

```yaml
cyberark_sia_policy_targets:
  AWS:
    accountIds: []      # empty = all accounts
    vpcIds: []
    regions: [eu-west-1, eu-west-2]
    tags:
      - key: Environment
        value: [production]
```

### VM — FQDN/IP (on-prem)

```yaml
cyberark_sia_policy_targets:
  FQDN/IP:
    fqdnRules:
      - operator: WILDCARD   # WILDCARD | EXACTLY | PREFIX | SUFFIX | CONTAINS
        computernamePattern: "*.example.internal"
        domain: "example.internal"
    ipRules:
      - logicalName: "prod-web-01"
        operator: EXACTLY
        ipAddresses:
          - "10.10.1.100"   # exact IPs only — CIDR and wildcards are not accepted
```

### Cloud Console — AWS IAM

```yaml
cyberark_sia_policy_targets:
  targets:
    - roleId: arn:aws:iam::123456789012:role/MyRole
      workspaceId: "123456789012"
```

### Entra Group assignment

```yaml
cyberark_sia_policy_targets:
  - groupId: c63819e2-2397-4faa-850f-4abde34e52fb
    directoryId: 280a06f4-3f9b-4910-8967-053a914e314e
```

## Conditions

```yaml
cyberark_sia_policy_conditions:
  accessWindow:
    daysOfTheWeek: [1, 2, 3, 4, 5]   # 0=Sun, 1=Mon … 6=Sat
    fromHour: "08:00"                  # HH:MM — seconds not accepted
    toHour: "18:00"
  maxSessionDuration: 4                # hours
```

`idleTime` (minutes) is set via `cyberark_sia_policy_idle_time` and merged into conditions automatically for VM and DB policies.

## Access approval (OnDemand policies)

Requires the Access Approval feature to be enabled in the tenant.

```yaml
cyberark_sia_policy_entitlement:
  policyType: OnDemand

# Do NOT include accessWindow when using accessApproval — the API rejects both together.
cyberark_sia_policy_conditions:
  maxSessionDuration: 2

cyberark_sia_policy_access_approval:
  required: true
  approvers:
    - id: 83170c32_8376_4358_8554_da221db3d9e1
      name: "SIA - Access Approvers"
      sourceDirectoryName: Identity
      sourceDirectoryId: Identity
      type: ROLE
```

**API constraint:** all `fqdnRules` in an approval policy must use `operator: EXACTLY` — wildcard, prefix, suffix, and contains operators are rejected.

## Iterating over multiple users

Use `policy_definitions | product(target_users) | list` to stamp out a consistent policy set for every user without repeating role invocations:

```yaml
vars:
  _sia_policy_state: present   # staging var — avoid self-reference when passed into role

  target_users:
    - username: john.smith@example.com
      display_name: "John Smith"
    - username: jane.doe@example.com
      display_name: "Jane Doe"

  policy_definitions:
    - name_suffix: "On-Prem SSH (Domain Wildcard)"
      entitlement: {targetCategory: VM, locationType: FQDN/IP, policyType: Recurring}
      conditions:
        accessWindow: {daysOfTheWeek: [1,2,3,4,5], fromHour: "08:00", toHour: "18:00"}
        maxSessionDuration: 4
      idle_time: 30
      targets:
        FQDN/IP:
          fqdnRules:
            - {operator: WILDCARD, computernamePattern: "*.example.internal", domain: "example.internal"}
          ipRules: []
      behavior:
        connectAs: {ssh: {username: svc_sia_linux}}
      tags: [linux, onprem, ssh]
      description: "SSH to all Linux hosts in example.internal"

tasks:
  - name: Apply SIA policy
    include_role:
      name: cyberark_sia_policy_management
    vars:
      cyberark_sia_policy_name: "{{ item[1].display_name }} - {{ item[0].name_suffix }}"
      cyberark_sia_policy_description: "{{ item[0].description }} for {{ item[1].display_name }}"
      cyberark_sia_policy_state: "{{ _sia_policy_state }}"
      cyberark_sia_policy_tags: "{{ item[0].tags }}"
      cyberark_sia_policy_entitlement: "{{ item[0].entitlement }}"
      cyberark_sia_policy_principal_usernames:
        - "{{ item[1].username }}"
      cyberark_sia_policy_conditions: "{{ item[0].conditions }}"
      cyberark_sia_policy_idle_time: "{{ item[0].idle_time | default(10) }}"
      cyberark_sia_policy_targets: "{{ item[0].targets }}"
      cyberark_sia_policy_behavior: "{{ item[0].behavior }}"
      cyberark_sia_policy_access_approval: "{{ item[0].access_approval | default(omit) }}"
    loop: "{{ policy_definitions | product(target_users) | list }}"
    loop_control:
      label: "{{ item[1].display_name }} / {{ item[0].name_suffix }}"
```

See [`examples/onprem_linux_ssh_multi_user.yml`](examples/onprem_linux_ssh_multi_user.yml) for the full working example.

## Exposed facts

After the role runs, the following facts are available to downstream tasks:

| Fact | Description |
|---|---|
| `cyberark_sia_policy_id` | The policy UUID |
| `cyberark_sia_policy_detail` | Full policy object — populated on both create and update (GET-after-PUT on update path) |
| `cyberark_sia_policy_principals` | Resolved principal list (set after Identity lookup) |

## Examples

| File | Description |
|---|---|
| [`site.yml`](examples/site.yml) | Single VM/AWS policy — principal resolved from vaulted account username |
| [`multiple_policies.yml`](examples/multiple_policies.yml) | VM, DB, Cloud Console, and Groups policies using Identity roles as principals |
| [`onprem_linux_ssh.yml`](examples/onprem_linux_ssh.yml) | On-prem Linux SSH — FQDN wildcard, exact hosts, and on-demand with approval |
| [`onprem_linux_ssh_multi_user.yml`](examples/onprem_linux_ssh_multi_user.yml) | User × policy matrix — consistent policy set applied across multiple users |
| [`policy_operations.yml`](examples/policy_operations.yml) | Get all, get one, create, and update operations |

## Removing policies

Set `cyberark_sia_policy_state: absent` to delete a policy by name.

## License

MIT
