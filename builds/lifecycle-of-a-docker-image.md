# Lifecycle of a Docker image

This document explains how a commit to the platform repo eventually becomes a new Docker image in master.

The process will inevitably change over time, so think of this more as a series of pointers that you can use to work out the current process, not a complete description.

### 1. A developer pushes code to master

### 2. This starts a build in Travis CI

The Travis build is defined by `.travis.yml` (see version at [d84e15c](https://github.com/wellcometrust/platform/blob/d84e15c5dcec49300d9eebfe640bed3c725112f9/.travis.yml#L139)).

Each app is built by a single job, which is defined as an entry in the `jobs` array.
The `TASK` environment variable tells us which app is built by this job.

Here's a cutdown example that builds three apps: `api`, `ingestor` and `transformer`.
See [L93-181 in the real file](https://github.com/wellcometrust/platform/blob/d84e15c5dcec49300d9eebfe640bed3c725112f9/.travis.yml#L93-L181).

```yml
jobs:
  include:
    - env: TASK=api-test
    - env: TASK=ingestor-test
    - env: TASK=transformer-test
```

Then Travis runs whatever is defined by the `script` block.
In [L76-77](https://github.com/wellcometrust/platform/blob/d84e15c5dcec49300d9eebfe640bed3c725112f9/.travis.yml#L76-L77):

```yml
script:
  - ./run_travis_task.py
```

### 3. Travis runs `run_travis_task.py` with a `TASK` environment variable

See version at [d84e15c](https://github.com/wellcometrust/platform/blob/d84e15c5dcec49300d9eebfe640bed3c725112f9/run_travis_task.py).

Start by reading the `main()` function, which is the main runner.

1.  First, we call `_should_run_tests()` to check if we can skip running builds.

    We have logic that checks the changed files in a pull request/push to decide if we can skip the build (e.g. if the only files that have changed are associated with the ingestor, we don't need to run an API build).

    TODO: When we've broken up our monorepo into smaller pieces, we can ditch this code, and run everything every time.
    It's fiddly and potentially a source of bugs.

2.  If we have changes that require a build, we call `unpack_secrets()`.

    This unpacks the encrypted ZIP bundle containing our Travis secrets.
    TODO: Explain how to build a new version of this bundle.

3.  Then we call `make(task)`, which is a wrapper around

    ```console
    $ make $TASK
    ```

    So we might run `make("ingestor-test")`, which runs the ingestor tests.

4.  If the tests pass, we check if this test task has a corresponding publish task, and construct the name of it if so.
    For example, `ingestor-test` becomes `ingestor-publish`.

5.  If we're running in a push to master (i.e. not a pull request or cron job), we call `make(publish_task)`, which is a wrapper around something like

    ```console
    $ make ingestor-publish
    ```

### 4. We call `make app-publish` for the specified app

Our Makefiles themselves are pretty fiddly, and worth a separate document.
(TODO: Write about our Makefiles.)

In brief, the interesting code is in [functions.Makefile](https://github.com/wellcometrust/platform/blob/5584c14a12ff55f64d921f5ff2ce2f34d4be68d9/functions.Makefile#L216-L217).

The task is defined as part of a series of tasks for a Scala service, in L203–18:

```make
# Define a series of Make tasks (build, test, publish) for a Scala services.
#
# Args:
#	$1 - Name of the project in sbt.
#	$2 - Root of the project's source code.
#
define __sbt_target_template
$(eval $(call __sbt_base_docker_template,$(1),$(2)))

$(1)-build:
	$(call sbt_build,$(1))
	$(call build_image,$(1),$(2)/Dockerfile)

$(1)-publish: $(1)-build
	$(call publish_service,$(1))
endef

```

Let's explain what happens when you run `make ingestor-publish`:

*   Because `$(1)-build` is a dependency of `$(1)-publish`, Make starts by running `make ingestor-build`.

*   `ingestor-build` starts by calling another Make function, `sbt_build`, which is defined on L164–174:

    ```make
    # Build an sbt project.
    #
    # Args:
    #   $1 - Name of the project.
    #
    define sbt_build
      $(ROOT)/docker_run.py --dind --sbt --root -- \
        --net host \
        wellcome/sbt_wrapper \
        "project $(1)" stage
    endef
    ```

    This calls `docker_run.py`, which is a wrapper for `docker run` with some convenience arguments.
    Here it's sharing the sbt directories with the container (`--sbt`) and passing in the entire repo (`--root`).
    This in turn calls the *wellcome/sbt_wrapper* image, which is defined in the [dockerfiles repo](https://github.com/wellcometrust/dockerfiles/blob/8391ef235e93ed0815abae6100a717b86c70a76f/sbt_wrapper).

    This image is a thin wrapper around sbt, and expands to something like:

    ```console
    $ sbt "project ingestor" stage
    ```

    The `stage` command is provided by the [sbt-native-packager plugin](https://sbt-native-packager.readthedocs.io/en/stable/archetypes/java_app/index.html), and builds a compiled version of the app in the `target` directory.

    TODO: Do we need `--dind` or `--net host`?

*   `ingestor-build` then calls a second Make function, `build_image`, which is defined on L106–18:

    ```make
    # Build and tag a Docker image.
    #
    # Args:
    #   $1 - Name of the image.
    #   $2 - Path to the Dockerfile, relative to the root of the repo.
    #
    define build_image
    	$(ROOT)/docker_run.py \
    	    --dind -- \
    	    wellcome/image_builder:latest \
                --project=$(1) \
                --file=$(2)
    endef
    ```

    which runs the [*wellcome/image_builder* image](https://github.com/wellcometrust/dockerfiles/blob/dd4f1f7215558cf876ee706b9b1d8b4130d0d7fd/image_builder).

    This image is a thin wrapper around `docker build`.
    It builds a new Docker image with the compiled code, tags the image with the current commit, and saves the commit ID to the `.releases` directory in the root of the repo.

    So now we have an image tagged `ingestor:123...def`.

*   Now `ingestor-build` is finished, the `publish` task kicks in.
    This calls a third Make function, `publish_service`, which is defined on L57–70:

    ```make
    # Publish a Docker image to ECR, and put its associated release ID in S3.
    #
    # Args:
    #   $1 - Name of the Docker image.
    #
    define publish_service
    	$(ROOT)/docker_run.py \
    	    --aws --dind -- \
    	    wellcome/publish_service:latest \
    	        --project="$(1)" \
    	        --namespace=uk.ac.wellcome \
    	        --infra-bucket="$(INFRA_BUCKET)" \
    			--sns-topic="arn:aws:sns:eu-west-1:760097843905:ecr_pushes"
    endef
    ```

    This runs the [*wellcome/publish_service* image](https://github.com/wellcometrust/dockerfiles/blob/3998ad48383879e9b3d5bedb52ccef66da9778b6/publish_service), which takes the name of the project, works out the ECR repo URI, and pushes the image to ECR.
