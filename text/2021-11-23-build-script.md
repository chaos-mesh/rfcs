# Build Script

## Summary

The Makefile has become too complicated and hard to understand, while we
actually need the reusablity and customizability. In this rfc, I will talk about
the design of Chaos Mesh build script, and a plan to migrate from current
`Makefile` to the build script.

## Background

There are three different roles who will use our build system:

1. Github Action
2. Jenkins CI
3. Developers

They have different requirements for the build script. The following section
will describe the difference between them, and what it means for our build system.

### Github Action

Github action is now widely used to run integration test, build multi-platform
package and publish them on the `ghcr.io`. The github action runs on `ubuntu`
virtual machine and other container images. It has several significant
advantages:

1. The docker daemon experimental feature has been turned on, which means we can
   use `docker buildx` and `--platform` arguments to build for different
   platforms.
2. There is a K-V cache system in github action. The cache can be dumped
   according to a given path, and be used the next time:
   ```yaml
   - name: Cache multiple paths
     uses: actions/cache@v2
     with:
       path: |
         ~/cache
         !~/cache/exclude
       key: ${{ runner.os }}-${{ hashFiles('**/lockfiles') }}
   ```
3. In the future, we hope to provide an action to publish image for PR, to help
   others to deploy the demo. The current problem is that reusing github action
   yaml is difficult and hard to manage. Another problem is that the cross
   building based on simulation is time consuming. Costing 20-40 * n minutes on
   every PRs is not realistic (which means, real cross build is required)

### Jenkins CI

Without asking the team who manages the Jenkins CI in PingCAP for help to modify
configuration, we can do little on the current situation of Jenkins environment.
Chaos Mesh teams are using Jenkins CI to run the e2e test, build nightly and
release x86 image and push them to `docker.io/pingcap`. It has the following
problems:

1. The docker daemon experimental feature is off. The `--platform` argument is
   invalid and will throw an error.
2. The cache is hard to manage. We are packaging the cache inside the e2e base
   image now, and the cache is update manually (by YangKeao).
3. The publish of image has been run on a release node, which means we will
   publish the latest image to `hub.pingcap.net` (an internal registry), and
   then republish them to the `docker.io` registry on release node.

### Developer

The developers on different platforms will use different tools to organize their
development environment. They may run the development cluster in minikube, kind
or other kubernetes cluster, and the linux environment could be a virtual
machine, a docker image or a host linux. Then, the build script needs the
following functions:

1. The developers could build image with customized `PROJECT` and `REGISTRY`.
   For example, YangKeao may want to build and upload the images to
   `hub.pingcap.net/yangkeao/chaos-mesh:latest`, in which the `PROJECT` is
   `yangkeao`, and `REGISTRY` is `hub.pingcap.net`.
2. The cache should behave well. The build and run loop should be really
   efficient.
3. It would be better to provide some helpers for debug, like a script to run
   chaos controller on a local machine which can manage a "remote" cluster. (as
   the remote delve debug function is not stable enough.)
4. It should help the developers to setup environment for e2e tests, run e2e
   tests, and do some cleaning jobs. It should also help the developers to
   format codes, generate files and run lints.
5. Setting up some fancy git pre-commit hook would also be great: like checking
   the DCO sign off. Fixing the sign off after merging and rebasing is really a
   nightmare.

## Function

We should also notice that: the `Dockerfile`, `Makefile`, and github action file
will also exist in this project. The responsibility of them and the "Build
Script" should be defined carefully.

What the **Build Script** should do:

1. Prepare build environemnt. In Chaos Mesh, it means to check and download the
   image containing all tools needed to construct Chaos Mesh.
2. Build final and middle targets, like an image, a binary or generating files.
3. Manage the build cache. Every building should leave some cache to accelerate
   the next building.
4. **Define and check the depencency relationship**. It has to be done by the
   build script, because all artifacts of `Makefile` are files, which means it
   knows nothing about the docker image. However, it's quite common to depend on
   and image.

   The Chaos Mesh uses `.dockerbuilt` file to represent the image has been
   built. However, it doesn't work well when the user switch the target
   IMAGE_TAG, REGISTRY ...
5. **Manage different kinds of building**. Some dependencies needs to be built
   directly, some need to be downloaded, while some others need `docker pull`.
6. Some helper script to upload images to specific registry.
7. Some helper script to run the e2e test, integration test, ...
8. Some helper to manage multiarch building, specifying `--platform` arguments
   for docker and set correct `TARGET_PLATFORM` arguments in `Dockerfile`.
   Parsing `uname -m` should also be its task.

What the **Makefile** should do:

1. Provide a convinient entrance to the build script. For example: `make
   image-chaos-mesh`, `make images/chaos-mesh/bin/chaos-controller-manager`...

What the **Dockerfile** should do:

1. Prepare the runtime (and runtime dependencies) for every components.
2. Pack the binary into the image

## Implementation

To import a standalone build script, we need to do some modifications to the
existing construction system:

Makefile:

1. The `Makefile` should have only several entries:
   - `help`: print the help message. It is the **default** target of the `Makefile`.
   - `image`: it will build the `chaos-mesh`, `chaos-daemon`, `chaos-dashboard`,
     `chaos-kernel`... images. Most of the time, `chaos-kernel`, `build-env` and
     `dev-env` will be ignored (unless flag `BUILD_IMAGE_KERNEL`... is set).
   - `image-*`: e.g. `image-chaos-daemon`, it will build the `chaos-daemon` image.
      It will also include `image-build-env` and `image-dev-env`.
   - binary under `images`, e.g. `images/chaos-mesh/bin/chaos-controller-manager`
   - `yaml`: generate the yaml files (e.g. CRD).
   - `install.sh`: generate the `install.sh`.
   - `generate`: run `chaos-builder`, and generate the deepcopy related files.
   - `test`: means to do some preparation and run the `go test`.
   - `e2e-test`: run the e2e test. **By default**, it will use the existing
     cluster, and will **not** install the chaos mesh. There are two flags:
     `E2E_TEST_CREATE_CLUSTER` and `E2E_TEST_INSTALL_CHAOS_MESH` to controll
     the corresponding behavior.
2. **Every** targets of `Makefile` is `.PHONY`, because the dependency will be
   controlled by the build scripts.


Dockerfile:

Don't need to be modified. But we should reconsider the relationship between
`build-env` and `dev-env`. I cannot tell why there are two similar images :(

### Interface Design

Then we need to talk about the interface of the build script. Assume the build
script is called `build`, then it will have nearly the same interface as the
`Makefile`. The `Makefile` will only turn the `make` into `./build`. For
example:

`make image-chaos-mesh` will turn into `./build image-chaos-mesh`. The `make
image` will also turn into `./build image`. The extra arguments are also passed
through environment variable, but in a well documented way.

Every targets in the build scripts will have following functions:

1. Return the modified time, to help others to assert whether their dependencies
   have updated.
2. Run or DryRun the build process.
3. Return the help message.

Here are some examples of one "target" and their behavior in the build script:

1. `./cmd/controller-manager/main/go`:
   - The modified time can be got from the file stats.
   - It will run nothing.
   - It doesn't have any help message.
2. `image-build-env`:
   - The modified time can be got from the `docker` command.
   - The build script should compare the modified time of this image with the
     modified time of files under `images/build-env`. If the `Dockerfile` is
     newer, it will download the image from the registry.
3. `image-chaos-mesh`:
   - The modified time can be got from the `docker` command.
   - The build script will build the dependent binaries automatically. Then it
     will compare the modified time of its dependencies and itself to decide
     whether it should be rebuilt.
4. `images/chaos-mesh/bin/chaos-controller-manager`:
   - The modified time can be got from the file stats;
   - It will enter the `build-env` and run `go build` to build the binary.
5. `test`:
   - Nothing should get its modified time.
   - It will enter the `dev-env` and run `go test` to run the test.

### Technical Implementation

In 2021, we can assume that the machine of every developer should have installed
the `python`, so we can use `python` to implement the build script. Targets with
different building behaviors can be concluded into different classes:

- `BaseTarget`: it will provide helper functions to verify and construct the
  dependencies, which is nearly needed by all targets.
   - `DockerImageTarget`: it will provides methods to get the modified time of
     the image, and to build the image.
   - `FileTarget`: it will provide method to get the modified time of a file.
      - `GoBuildTarget`: it will provide method to get the modified time of the
      binary and to build the binary.
      - ... Some other targets to represent the generated files.
- `CollectionTarget`: like the `image` target, it will build all targets inside.
   It will **not** provide or verify the modification time, because the targets
   inside will handle the dependencies well.

Then we also need some helpers to execute commands and redirect the output.

- `ImageRunner`: it will run a command in the docker image.
- `RawCommandRunner`: it will run a command directly.

These command runners should accept some arguments (or env variables) to decide
whether to run in dry-run mode.

## Imagination

- If the `Makefile` targets are only an entry point to the build script, can it
  be generated directly from the build script?
- Is `python` a good choice? Is our building environment having a updated `python`?
- I have considered `google/tx` and `bash` script as an alternative to `python`.
  From the personal feeling, I prefer nodejs (with `google/tx`), and it has much
  better performance than `python` (considering we may need to read a lot of
  files stats and do a lot of asynchronous job). However, it's not widely
  accepted. 
  
  And I don't have much confidence to write a `bash` script to handle the
  dependencies, which is really needed to be done in build script.
- Parallel building. Is it possible to build the dependencies in parallel?