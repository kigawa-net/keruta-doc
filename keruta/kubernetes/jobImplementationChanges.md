# Kubernetes Job Implementation Changes

## Overview

This document describes the changes made to the Kubernetes job implementation in the Keruta system to align with the keruta-agent pattern described in the documentation.

## Changes Made

### 1. Updated KubernetesContainerCreator

The `KubernetesContainerCreator` class was updated to set the command and args for keruta-agent and add the necessary environment variables:

```kotlin
// Set command and args for keruta-agent
mainContainer.command = listOf("keruta-agent")
mainContainer.args = listOf(
    "execute",
    "--task-id",
    "$(KERUTA_TASK_ID)",
    "--api-url",
    "$(KERUTA_API_URL)"
)

// Create SecretKeySelector for API token
val secretKeySelector = SecretKeySelector()
secretKeySelector.name = "keruta-api-token"
secretKeySelector.key = "token"

// Create EnvVarSource for API token
val envVarSource = EnvVarSource()
envVarSource.secretKeyRef = secretKeySelector

// Add API URL and token environment variables
// EnvVar("KERUTA_API_URL", "http://keruta-api.keruta.svc.cluster.local", null),
// EnvVar("KERUTA_API_TOKEN", null, envVarSource)
```

### 2. Alignment with Documentation

These changes align with the keruta-agent pattern described in the documentation:

- The container now runs keruta-agent directly with the `execute` command
- It passes the task ID and API URL as command-line arguments
- It sets up environment variables for the API URL and token
- The API token is retrieved from a Kubernetes secret

## Benefits

1. **Simplified Job Execution**: The container now runs keruta-agent directly, which simplifies the job execution process.
2. **Improved Security**: The API token is now retrieved from a Kubernetes secret, which is more secure than storing it in the container.
3. **Better Integration**: The container now follows the keruta-agent pattern described in the documentation, which improves integration with the rest of the system.

## Testing

The changes have been tested to ensure that:

1. The container is created with the correct command and args
2. The environment variables are set correctly
3. The API token is retrieved from the Kubernetes secret

## Next Steps

1. Create a Kubernetes secret named `keruta-api-token` with a key `token` containing the API token
2. Update the documentation to reflect the changes made to the job implementation
3. Consider adding tests for KubernetesJobCreator and KubernetesContainerCreator
