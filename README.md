# containers-service-update

GitHub composite action to update an existing DTZ Containers service. The action first fetches the current service, overlays any provided inputs, and posts the update. Only `api_key` and `service_id` are required; all other fields are optional.

- API base URL is fixed to `https://containers.dtz.rocks`
- API version is fixed to `2021-02-21`

## Usage

```yaml
- uses: DownToZero-Cloud/containers-service-update@main
  with:
    api_key: ${{ secrets.DTZ_API_KEY }}
    service_id: service-abc123
    enabled: true
    prefix: /app
    env_variables: '{"LOG_LEVEL":"debug","SECRET":{"plainValue":"s3cr3t"}}'
```

### Second example: Update the image URL (repository + tag)

```yaml
- uses: DownToZero-Cloud/containers-service-update@main
  with:
    api_key: ${{ secrets.DTZ_API_KEY }}
    service_id: service-abc123
    container_image: myregistry/webapp@sha256:abc3534
```

## Inputs

- `api_key` (required): DTZ API key (`X-API-KEY`).
- `service_id` (required): ServiceId to update.
- `enabled` (optional): `true` or `false`. Omit to keep as-is.
- `domain` (optional): Comma-separated list of domains. Set to empty string `""` to clear. Omit to keep as-is.
- `prefix` (optional): URL prefix, e.g. `/` or `/app`.
- `container_image` (optional): Container image repo, e.g. `nginx` or `myrepo/app`.
- `container_image_version` (optional): Image tag or digest, e.g. `:v1.2.3` or `@sha256:...`.
- `container_pull_user` (optional): Registry username.
- `container_pull_pwd` (optional): Registry password.
- `env_variables` (optional): JSON object of env vars. Example: `{"KEY":"value","SECRET":{"plainValue":"x"}}`.
  - Supports plain strings, `EncryptedValue` and `PlainValue` shapes as per the API spec.
- `rewrite_source` and `rewrite_target` (optional, pair): Set URL rewrite. Both must be provided.
- `login_provider_name` (optional): Set to e.g. `dtz`. Set to empty string `""` to remove login. Omit to keep as-is.

Notes:
- If an optional input is omitted, the current value from the fetched service is kept.
- Setting `domain: ""` results in an empty domain array (no domains).
- `enabled` is parsed case-insensitively; only the string `"true"` becomes true when provided.

## Outputs

- `service`: Compact JSON string of the updated service object returned by the API.

Example to capture the updated service:

```yaml
- id: update
  uses: DownToZero-Cloud/containers-service-update@main
  with:
    api_key: ${{ secrets.DTZ_API_KEY }}
    service_id: abc123
- name: Show updated service
  run: |
    echo '${{ steps.update.outputs.service }}' | jq .
  shell: bash
```

## More Examples

- Update container image and pin to digest:

```yaml
- uses: DownToZero-Cloud/containers-service-update@main
  with:
    api_key: ${{ secrets.DTZ_API_KEY }}
    service_id: abc123
    container_image: myregistry/secure-app
    container_image_version: '@sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7'
```

- Clear domains and remove login:

```yaml
- uses: DownToZero-Cloud/containers-service-update@main
  with:
    api_key: ${{ secrets.DTZ_API_KEY }}
    service_id: abc123
    domain: ""
    login_provider_name: ""
```

- Set URL rewrite (source regex to target):

```yaml
- uses: DownToZero-Cloud/containers-service-update@main
  with:
    api_key: ${{ secrets.DTZ_API_KEY }}
    service_id: abc123
    rewrite_source: '^/old/(.*)$'
    rewrite_target: '/new/$1'
```

- Update only environment variables:

```yaml
- uses: DownToZero-Cloud/containers-service-update@main
  with:
    api_key: ${{ secrets.DTZ_API_KEY }}
    service_id: abc123
    env_variables: '{
      "LOG_LEVEL":"info",
      "DATABASE_URL":{"plainValue":"postgresql://user:pwd@host:5432/db"}
    }'
```

## How it works

1. Fetches current service: `GET /api/2021-02-21/service/{serviceId}`
2. Builds an overlay from provided inputs.
3. Merges overlay over fetched service to ensure required fields remain present.
4. Updates service: `POST /api/2021-02-21/service/{serviceId}`

## Requirements

- Runs on Bash with `curl` and `jq` available (present on GitHub-hosted Ubuntu runners).

## Security

- Always pass `api_key` via GitHub Encrypted Secrets, e.g. `${{ secrets.DTZ_API_KEY }}`.