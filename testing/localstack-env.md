# LocalStack Environment Configuration

When working with LocalStack in this project, always include the following environment detection code at the beginning of any bash script that uses the `awslocal` command:

```bash
# Environment detection for LocalStack endpoint
if [ "${LOCALSTACK_DEV_CONTAINER}" = "true" ]; then
    BASE_URL="http://host.docker.internal:4566"
else
    BASE_URL="http://localhost:4566"
fi
export AWS_ENDPOINT_URL="${BASE_URL}"
```

This configuration is necessary because:
1. When running in a devcontainer, LocalStack must be accessed via `host.docker.internal`
2. When running directly on the host machine, LocalStack is accessed via `localhost`
3. The AWS endpoint URL must be properly set for `awslocal` commands to work correctly

Key points to remember:
- Always include this environment check in LocalStack-related scripts
- Set `LOCALSTACK_DEV_CONTAINER=true` when working in the devcontainer
- Use `awslocal` commands after setting the endpoint URL
- Verify the endpoint with `echo "Using endpoint: ${AWS_ENDPOINT_URL}"` after setting it

Example usage in scripts:
```bash
#!/bin/bash

# Environment detection for LocalStack endpoint
if [ "${LOCALSTACK_DEV_CONTAINER}" = "true" ]; then
    BASE_URL="http://host.docker.internal:4566"
else
    BASE_URL="http://localhost:4566"
fi
export AWS_ENDPOINT_URL="${BASE_URL}"

echo "Using endpoint: ${AWS_ENDPOINT_URL}"
# Your LocalStack commands here
```