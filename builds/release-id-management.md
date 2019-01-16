# Release ID management

This document explains how the ID of a Docker image eventually becomes part of a task definition in ECS.

Docker images are built automatically by Travis CI (see [Lifecycle of a Docker image](Lifecycle of a Docker image)).


## Use case

If we don't pin Docker images in our ECS task definitions, then ECS would pull the latest Docker image from ECR -- even if that's changed since the task definition was created.
This means the version of our code which is deployed could change without us realising!

We pin Docker images in our task definitions so that our code doesn't change without an explicit step from us (running a `terraform apply`).


## Process

### 1. Travis CI builds a Docker image

This is handled by the `wellcome/publish_service` image.

This creates the image ID (typically the Git commit hash, so we can easily trace an image back to the code that built it).

It uploads the complete image URL (ECR repository URL and image ID) to Amazon Parameter Store.

### 2. We run `terraform plan`.

tbc
