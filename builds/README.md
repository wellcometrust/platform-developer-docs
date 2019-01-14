# builds

Here "builds" means anything that happens automatically -- builds, tests, publishing new code.
The platform build system has grown organically, and that means it's quite fiddly and complicated in parts.
This README and associated docs tries to explain some of the thinking behind it.

![Two men in sewer tunnels, one holding a lantern and looking at some pipes](sewage.jpg)

<sub>Two developers try to understand the platform build system. <em>&ldquo;And where does this part lead?&rdquo;</em> Image: <a href="https://wellcomecollection.org/works/j3hbjmjc">Wellcome Collection</a>, CC BY.</sub>



## Everything is a Docker image

Almost everything we do runs inside Docker containers.
All the usual benefits of Docker apply: reproducibility, portable builds, reduced local dependencies.

All our Docker images live in the [wellcometrust/dockerfiles](https://github.com/wellcometrust/dockerfiles) repo, and they get automatically published to the [wellcome organisation on Docker Hub](https://hub.docker.com/u/wellcome).

If you're trying to work out what a particular Docker image does, looking in that repo is a good place to start.


## The lifecycle of a Docker image

All our platform apps are published as Docker images in ECR.

For a walkthrough of how that works, see the [lifecycle of a Docker image](lifecycle-of-a-docker-image.md).
