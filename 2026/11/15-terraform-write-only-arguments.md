# Keeping Secrets Out of Terraform State with Write-Only Arguments

*Cloud & infrastructure-as-code · 2026-11-15*

## What it is

Since Terraform 1.11, providers can mark resource arguments as *write-only* (by convention suffixed `_wo`). Terraform hands the value to the provider during the operation and then drops it: it never lands in the plan file or `terraform.tfstate`. Paired with *ephemeral* resources (1.10+), which also never persist, you can generate or fetch a secret and push it into a resource without the plaintext ever touching state.

## Example

```hcl
ephemeral "random_password" "db" {
  length = 24
}

resource "aws_db_instance" "app" {
  engine              = "postgres"
  instance_class      = "db.t3.micro"
  allocated_storage   = 20
  username            = "app"
  skip_final_snapshot = true

  password_wo         = ephemeral.random_password.db.result
  password_wo_version = 1
}
```

- `ephemeral "random_password"` generates the value each run and keeps it out of state (unlike `resource "random_password"`, whose `result` is stored in plaintext).
- `password_wo` is the write-only counterpart of `password`; the provider uses it, Terraform forgets it.
- `password_wo_version` is stored in state. Because Terraform can't diff a value it doesn't keep, bumping this number is the only way to trigger a rotation.

Verify: `terraform show -json | jq '.values.root_module.resources[] | select(.type=="aws_db_instance") | .values.password_wo'` should come back `null`.

## When to use it / why it matters

- **Defensive:** state files are a classic loot target. Pentesters regularly find `terraform.tfstate` in readable S3 buckets, CI artifacts or old Git history, full of DB passwords and API keys. `sensitive = true` only hides values from CLI output; they are still in state. Write-only args fix the actual storage problem.
- **Audit tip:** grep existing state for `"password":` / `"secret_string":` and migrate those resources to `password_wo`, `secret_string_wo` (Secrets Manager) or `value_wo` (SSM Parameter).
- **Gotchas:** the value is gone after apply, so store it somewhere you can read it, such as `aws_secretsmanager_secret_version.secret_string_wo`, or use RDS `manage_master_user_password` and let AWS own it. Changing the source value without bumping `*_wo_version` silently does nothing. Support depends on the resource and provider version, so check the registry docs. Your state backend still needs encryption and tight IAM; this fix covers secrets, not resource metadata.

## Sources

- [Terraform docs: Use write-only arguments](https://developer.hashicorp.com/terraform/language/resources/ephemeral/write-only)
- [HashiCorp blog: Terraform 1.11 brings ephemeral values to managed resources](https://hashicorp.com/blog/terraform-1-11-ephemeral-values-managed-resources-write-only-arguments)
- [Terraform docs: Sensitive data in state](https://developer.hashicorp.com/terraform/language/state/sensitive-data)
