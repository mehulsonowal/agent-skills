# Provider: AWS Bedrock

Triggered by introspection (Workflow step 2.5) when the call-site function imports `boto3` and calls:

- `boto3.client("bedrock-runtime").invoke_model(...)`
- `boto3.client("bedrock-runtime").converse(...)` (newer API)
- `boto3.Session(...).client("bedrock-runtime")...`

## `{{PROVIDER_ASSERTS}}` substitution

Only emit static credential asserts when the call-site explicitly supplies access-key credentials (for example,
`aws_access_key_id=`/`aws_secret_access_key=` kwargs or bare env-var usage without a profile, session, or role).
Otherwise trust boto3's default credential provider chain and emit a note instead: profiles, SSO, IAM Identity Center,
and EC2/ECS/Lambda roles do not require these environment variables.

```python
assert os.getenv("AWS_ACCESS_KEY_ID"), "AWS_ACCESS_KEY_ID is required for the wired task_fn (Bedrock)."
assert os.getenv("AWS_SECRET_ACCESS_KEY"), "AWS_SECRET_ACCESS_KEY is required for the wired task_fn (Bedrock)."
```

## Optional env vars

- `AWS_SESSION_TOKEN` — needed only when the selected credential source explicitly uses a session token. Do not
  require it for profile, SSO, IAM Identity Center, or instance-role credentials resolved by boto3.
- `AWS_REGION` (or `AWS_DEFAULT_REGION`) — Bedrock is region-scoped and boto3 does **not** provide a default region.
  If the call-site does not pass an explicit `region_name=`, emit:

  ```python
  assert os.getenv("AWS_REGION") or os.getenv("AWS_DEFAULT_REGION"), (
      "AWS_REGION (or AWS_DEFAULT_REGION) is required for Bedrock; "
      "boto3 has no default region."
  )
  ```

- `AWS_PROFILE` — an alternative credential source. If the user's function uses `boto3.Session(profile_name=...)`,
  skip static key-pair asserts and trust boto3's credential resolution.

## Adapter notes

- `client.invoke_model(modelId=..., body=...)` (older API) — body is a JSON string with provider-specific shape (Anthropic Claude, Amazon Titan, AI21, Cohere, Meta Llama — each has different body schema). Extract response via `json.loads(response["body"].read())`.
- `client.converse(modelId=..., messages=[...])` (newer Converse API) — standardized request/response across providers. Extract via `response["output"]["message"]["content"][0]["text"]`.
- If the user's function uses `invoke_model`, leave their body construction intact in `task_fn` — Anthropic-on-Bedrock vs Llama-on-Bedrock have different request shapes.

## Common gotchas

- Model IDs differ from upstream provider IDs (e.g., `anthropic.claude-3-5-sonnet-20240620-v1:0` rather than `claude-3-5-sonnet-20240620`). Don't rewrite the model ID.
- Bedrock charges per call; rate limits apply. Consider `--jobs 1` for the first experiment run to gauge cost.
- Cross-region inference profiles use a different model ID prefix (`us.anthropic...`, `eu.anthropic...`). Trust the user's setup.
