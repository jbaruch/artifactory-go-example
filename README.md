# Sample project to demonstrate Go language integration with JFrog Artifactory

## Overview
Go repositories are supported by Artifactory since version 5.11.0.
To work with Go repositories you need to use [JFrog CLI](https://www.jfrog.com/confluence/display/CLI/CLI+for+JFrog+Artifactory) and the Go client.

## Prerequisites
* Get Artifactory trial, [on-prem or in the cloud](https://jfrog.com/artifactory/free-trial/)
* Install version 1.11 or above of Go.
* Install version 1.19.1 of [JFrog CLI](https://jfrog.com/getcli/)
* Fork and clone this project

## Running the Example
### Create Go Repositories in Artifactory
* Click on *Quick Setup* under *Create Repositories* in your username drop-down menu in Artifactory UI.
* It will create a remote Go repository named *go-remote*, a local Go repository named *go-local*, and virtual Go repository named *go*, which will include the *go-remote* and *go-local*.

### Resolve Dependencies and Build the Project Using JFrog CLI
In the root directory of the project, perform the following steps:

1. Configure Artifactory. Follow the prompts of the `config` command:

`> jfrog rt config`

2. Once the JFrog CLI is configured with Artifactory, we can try and build our project, resolving the dependencies from Artifactory, hoping it will find the modules in GitHub using the remote repository proxying functionality (we'll use the *go*, which includes *go-remote*):

```> cd hello
> jfrog rt go build go
```

The build will probably error, failing to find one of the needed dependencies:
```> go: finding rsc.io/quote v1.5.2
> go: rsc.io/quote@v1.5.2: unexpected status (https://username@artifactory.jfrog.io/artifactory/api/go/go/rsc.io/quote/@v/v1.5.2.info): 404 Not Found
> go: error loading module requirements
> [Error] exit status 1
```

That makes sense, the packages modules do not exist in the remote repository (GitHub). No despair, we can package them ourselves, and serve from a local repository instead. Whoever builds after us, won't know the difference!

3. We'll switch to a "legacy" mode, in which Go will use the GitHub project sources instead of published modules. Let's build this project only once with the `--no-registry` option, to get the dependency *sources* from GitHub and let Go create modules out of them.

`> jfrog rt go build --no-registry`

4. Now that we fetched the project dependency sources from GitHub, let's create modules out of them and push them to Artifactory. As the previous command, this needs to be only done once. Once we published the modules to Artifactory, they are there for the other developers to use. We will use the virtual repository again, this time uploading files through it to *go-local*.

`> jfrog rt go-publish go --self=false --deps=ALL`

5. Now you can see the dependencies in Artifactory under *go-local* repository.

6. [Optional] to verify that you can use the modules from Artifactory now, you can delete the modules from the local cache (located at `$GO_PATH/pkg/mod`). In this case they will be resolved from Artifactory in the next step.

7. Build the project with Go and resolve the uncached dependencies from Artifactory. If you deleted the cache, you'll see the `go: downloading` messages when go downloads the missing dependencies from Artifactory. This will also declare the used dependencies in [Artifactory Build Integration](https://www.jfrog.com/confluence/display/RTF/Build+Integration) build-info metadata.

`> jfrog rt go build go --build-name=my-build --build-number=1`

### Create and publish a Go module to Artifactory
Depending on your scenario, you might want to package your source files as a Go module and publish it to Artifactory to be used as a dependency, by you, or other team members.

1. Publish the package we build to Artifactory. Build-info will record the uploaded module as an artifact

`> jfrog rt go-publish go v1.0.0 --build-name=my-build --build-number=1`

2. Collect environment variables and add them to the build-info

`> jfrog rt build-collect-env my-build 1`

3. Publish the build-info to Artifactory

`> jfrog rt build-publish my-build 1`

4. Now you can navigate to see the build in Artifactory by using the link in the console log.

### Create and publish a Go binary to Artifactory
Another (more common) use-case is to create a binary executable and publish it to Artifactory.

We recommend using the excellent [GoReleaser](https://goreleaser.com/) tool for that. Check the documentation of GoReleaser to install it.

Although GoReleaser has [a built-in support for Artifactory](https://goreleaser.com/customization/#Artifactory), we will use the JFrog CLI for actual deployment, and that to include the deployed artifacts in the build info metadata. (Note: pull request for build info support in GoReleaser is on its way)

1. First, you'll need a generic repository for the executables. Click Local Repository under *Create Repositories* in your username drop-down menu in Artifactory UI, and select Generic. We'll call it `binary-releases-local`.

2. Next, let's init GoReleaser

`> goreleaser init`

3. Now let's create the binaries. We'll run GoReleaser with `--snapshot` flag to generate the binaries, but skip the deployment.

`> goreleaser --snapshot`

4. Once the binaries are generated, let's deploy them to Artifactory with the build information

`> jfrog rt upload "dist/*.tar.gz" binary-releases-local --build-name=my-build --build-number=1`

5. Collect environment variables and add them to the build info

`> jfrog rt build-collect-env my-build 1`

6. Publish the build info to Artifactory

`> jfrog rt build-publish my-build 1`

7. Now you can navigate to see the build in Artifactory by using the link in the console log.
