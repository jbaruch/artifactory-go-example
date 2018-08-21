# Golang example

## Overview
Go repositories are supported by Artifactory since version 5.11.0.
To work with Go repositories you need to use [JFrog CLI](https://www.jfrog.com/confluence/display/CLI/CLI+for+JFrog+Artifactory) and the Go client.

## Prerequisite
* Install version 1.11 or above of Go (before Go 1.11 is released, please install one of its release candidates).
* Make sure the **go** command is in your PATH.
* Install [JFrog CLI](https://jfrog.com/getcli/)
* Make sure your Artifactory version is 5.11.0 or above
* Make sure your JFrog CLI version is 1.18.0 or above

## Running the Example
### Create Go Repositories in Artifactory
* Click on *Quick Setup* under *Create Repositories* in your username drop-down menu in Artifactory UI.
* It will create a remote Go repository named *go-remote*, a local Go repository named *go-local*, and virtual Go repository named *go*, which will include the *go-remote* and *go-local*.

### Resolve Dependencies and Build the Project Using JFrog CLI
CD to the root project directory

```console
Configure Artifactory:
> jfrog rt configure

First, let's try to build resolving the dependencies from Artifactory, hoping it will find the modules in GitHub using the remote repository proxing functionality (we'll use the *go*, which includes *go-remote*)
> jfrog rt go build go

The build will probably error, failing to find one of the needed dependencies:
> go: finding rsc.io/quote v1.5.2
> go: rsc.io/quote@v1.5.2: unexpected status (https://username@artifactory.jfrog.io/artifactory/api/go/go/rsc.io/quote/@v/v1.5.2.info): 404 Not Found
> go: error loading module requirements
> [Error] exit status 1

That makes sense, the packages modules do not exist in the remote repository (GitHub). No depare, we can package them ourselves, and serve from a local repository instead. Whoever builds after us, won't know the difference!

First build this project only once with the --no-registry option, to bypass Artifactory.
We're bypassing Artifactory to fetch the project dependencies from GitHub.
> jfrog rt go build --no-registry

Now that we fetched the project dependencies from GitHub, let's create modules out of them and push them to Artifactory. As the previous command, this needs to be only done once. Once we published the modules to Artifactory, they are there for the other developers to use. We will use the virtual repositoriy again, this time uploading files through it to *go-local*.
> jfrog rt go-publish go --self=false --deps=ALL

Now you can see the dependencies in Artifactory under *go-local* repository.
[Optional] to verify that you can use the modules from Artifactory now, you can delete the modules from the local cache (located at $GO_PATH/pkg/mod). In this case they will be resolved from Artifactory in the next step.

Build the project with go and resolve the uncached dependencies from Artifactory. If you deleted the cache, you'll see the *go: downloading* messages when go downloads the missing dependencies from Artifactory.
> jfrog rt go build go
```

### Create and publish a Go module to Artifactory
Depending on your scenario, you might want to package your source files as a Go module and publish it to Artficatory to be used as a dependency, by you, or other team members.

```Publish the package we build to Artifactory.
> jfrog rt go-publish go v1.0.0 --build-name=my-build --build-number=1

Collect environment variabkes and add them to the build info.
> jfrog rt build-collect-env my-build 1

Publish the build info to Artifactory.
> jfrog rt build-publish my-build 1
```

### Create and publish a Go binary to Artifactory
Another (more common) usecase is to create a binary executable and publish it to Artifactory. We recommend using the exellent [GoReleaser](https://goreleaser.com/) tool for that. Check the documentation of GoReleaser to get started and the [Artifactory](https://goreleaser.com/customization/#Artifactory) section to add the right deployment target. 

*//TODO add build-info to goreleaser artifacts**
