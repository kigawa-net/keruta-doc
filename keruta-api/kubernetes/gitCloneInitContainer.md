# Git Clone in Init Container

## Overview

Keruta now supports cloning Git repositories in Kubernetes init containers. This feature allows tasks to access code from Git repositories before the main container starts, enabling seamless integration with version-controlled code.

## How It Works

When a task is created with a reference to a Git repository, Keruta automatically:

1. Creates an empty directory volume named "repo-volume"
2. Adds an init container that clones the specified Git repository into this volume
3. Mounts the volume to the main container at `/repo`

This ensures that the Git repository is available to the main container when it starts.

## Usage

To use this feature:

1. Create a Repository in Keruta with the Git repository URL
2. When creating a Task, specify the Repository ID in the `repositoryId` field

The repository will be automatically cloned into the `/repo` directory in the container.

## Implementation Details

The implementation uses the `alpine/git` image to perform the Git clone operation. The init container runs a shell script that:

1. Creates the `/repo` directory
2. Changes to that directory
3. Clones the repository into the current directory
4. Outputs a success message

The repository is available to the main container at the `/repo` path.

## Example

```kotlin
// Create a repository
val repository = Repository(
    name = "Example Repository",
    url = "https://github.com/example/repo.git",
    description = "Example repository for demonstration"
)

// Create a task with the repository
val task = Task(
    title = "Task with Git Repository",
    description = "This task uses code from a Git repository",
    repositoryId = repository.id
)

// The repository will be cloned into /repo in the container
```

## Advanced Usage

For more advanced Git operations, you can customize the task's container image to include Git and perform additional operations in your application code.

## Limitations

- Currently, only public repositories or those that don't require authentication are supported
- Large repositories may take longer to clone, increasing the task startup time
- The repository is cloned fresh for each task execution