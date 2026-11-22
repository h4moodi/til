# Enforcing IMDSv2 by Default at the Account Level

*Cloud & infrastructure-as-code · 2026-11-22*

## What it is

IMDSv1 answers any plain `GET` to `169.254.169.254`, so one SSRF in an app on EC2 can hand over the instance role's credentials. IMDSv2 requires a session token fetched with a `PUT` first, which most SSRF primitives can't send. Instead of fixing every launch template, you can set *account-level, per-Region* defaults with `ModifyInstanceMetadataDefaults`, so new instances launch with tokens required unless something more specific overrides it.

## Example

```bash
# 1. Find instances still taking token-less calls (non-zero = IMDSv1 traffic)
aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name MetadataNoToken --dimensions Name=InstanceId,Value=i-0abc123 \
  --start-time 2026-11-15T00:00:00Z --end-time 2026-11-22T00:00:00Z \
  --period 86400 --statistics Sum

# 2. Make IMDSv2 the default for new launches in this Region
aws ec2 modify-instance-metadata-defaults --region eu-central-1 \
  --http-tokens required \
  --http-put-response-hop-limit 2

# 3. Check it
aws ec2 get-instance-metadata-defaults --region eu-central-1
```

- `MetadataNoToken`: per-instance CloudWatch metric counting IMDSv1 calls; drive it to zero before you enforce.
- `--http-tokens required`: token-less requests get `401`.
- `--http-put-response-hop-limit 2`: the token `PUT` response survives one extra network hop, needed when containers in bridge mode call IMDS. Use `1` where containers shouldn't reach IMDS at all (e.g. EKS nodes whose pods use IRSA or Pod Identity).
- It's per Region: loop over every Region you use.

## When to use it / why it matters

Defensive baseline for any AWS account. Gotchas: the defaults apply **only at launch**, so existing instances still need `modify-instance-metadata-options`. Settings in the launch template or `RunInstances` call, and AMIs registered with `ImdsSupport=v2.0`, take precedence. Account defaults are a default, not a guardrail: for that, use an SCP that denies `ec2:RunInstances` unless `ec2:MetadataHttpTokens` equals `required`, or an EC2 declarative policy in AWS Organizations. In Terraform, set `metadata_options { http_tokens = "required" }` on `aws_instance` / `aws_launch_template` so drift shows up in plans. On the offensive side, during authorized cloud tests, an SSRF that returns credentials from `/latest/meta-data/iam/security-credentials/` is a strong sign IMDSv1 is still on.

## Sources

- [Configure instance metadata options for new instances (AWS EC2 User Guide)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-IMDS-new-instances.html)
- [ModifyInstanceMetadataDefaults (EC2 API Reference)](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_ModifyInstanceMetadataDefaults.html)
- [Get the full benefits of IMDSv2 and disable IMDSv1 across your AWS infrastructure (AWS Security Blog)](https://aws.amazon.com/blogs/security/get-the-full-benefits-of-imdsv2-and-disable-imdsv1-across-your-aws-infrastructure)
