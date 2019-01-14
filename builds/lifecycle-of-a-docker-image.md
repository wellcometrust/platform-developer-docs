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
