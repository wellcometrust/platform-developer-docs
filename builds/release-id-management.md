# New release ID management

This document explains how the ID of a Docker image eventually becomes part of a task definition in ECS.

Docker images are built automatically by Travis CI (see [Lifecycle of a Docker image](Lifecycle of a Docker image)).


## Requirements

*   We want to know which version of each application is running at a particular time.
    This will allow us to debug failures and rollback a failed deployment.

*   We want to be able to run multiple environments (prod, stage, etc).


## Process


### 1) Build ~> artefact

CI or a developer builds a new Docker image locally.

The new Docker image is pushed to an ECR repository, and the complete ECR image URL is copied to SSM under the key:

    /{project_name}/image_ids/latest/{image_name}

TODO: none of our images contain secrets, so we should really be pushing them to Docker Hub.


### 2) Artefacts ~> release manifest

We usually deployed images in collections (for example, all the services in the catalogue pipeline).

We run a script that creates a *release manifest* describing a collection of Docker images that would be deployed together.

The manifest includes:

*   The image URLs, which are copied from SSM
*   A reason for this deployment
*   The date and user

The manifest is uploaded as a JSON file to S3 under the key:

    /{project_name}/release_manifests/{environment}/{version}.json

The version is monotonically increasing (v1, v2, v3, ...).

When we are deploying new images to the stage environment, the image URLs are copied from the `/latest/` keys in SSM.

When we're promoting images from staging to prod, the image URLs are copied from the `/stage/` keys in SSM.


### 3) Release manifest ~> deployment

When we have a release manifest, we run another script that copies the image URLs into SSM, under the key:

    /{project_name}/image_ids/{environment}/{image_name}

Terraform reads these keys, and uses them to create an ECS task definition.
We run a plan and apply, and the new task definition is deployed.

We also record that we are using the release manifest, and the time it was used.



## Discussion

### Rolling back to an older version

If we have a bad deployment, we can take the old release manifest and run it through step 3.
It gets a new version number.


### Selective updates

Currently step (2) promotes everything from latest to stage, or from stage to prod.

Later, we could modify this to selectively update image URLs -- i.e., only update a single version of an application.
