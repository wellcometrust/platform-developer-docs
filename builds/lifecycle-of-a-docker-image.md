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
