---
layout: implementors
title:  "Dev container Features contribution and discovery [proposal]"
shortTitle: "Features distribution"
author: Microsoft
index: 6
---

> Note: This section provides information on a currently active proposal. See the [Features distribution proposal in the spec repo](https://github.com/devcontainers/spec/issues/61) for input and links to other proposed improvements.

This specification defines a pattern where community members and organizations can author and self-publish [dev container 'features'](../features). 

Goals include:

- For community authors, a "self-served" mechanism for dev container feature publishing, either publicly or privately.
- For users, the ability to validate the integrity of previously fetched assets. 
- For users, the ability for a user to pin to a particular version (absolute, or semantic version) of a feature to allow for consistent, repeatable environments.
- The ability to standardize publishing such that [supporting tools](../../supporting) may implement mechanisms for feature discoverability.

## <a href="#source-code" name="source-code" class="anchor"> Source Code </a>

Features source code is stored in a git repository.

For ease of authorship and maintenance, [1..n] features can share a single git repository.  This set of features is referred to as a collection, and will share the same [`devcontainer-collection.json`](#devcontainer-collection.json) file and 'namespace' (eg. `<owner>/<repo>`).

Source code for the set follows the example file structure below:

```
.
├── README.md
├── src
│   ├── dotnet
│   │   ├── devcontainer-feature.json
│   │   ├── install.sh
│   │   └── ...
|   ├
│   ├── go
│   │   ├── devcontainer-feature.json
│   │   └── install.sh
|   ├── ...
│   │   ├── devcontainer-feature.json
│   │   └── install.sh
├── test
│   ├── dotnet
│   │   ├── test.sh
│   │   └── ...
│   └── go
│   |   └── test.sh
|   ├── ...
│   │   └── test.sh
├── ...
```

Where `src` is a directory containing a sub-folder with the name of the feature (e.g. `src/dotnet` or `src/go`) with at least a file named `devcontainer-feature.json` that contains the feature metadata, and an `install.sh` script that implementing tools will use as the entrypoint to install the feature.  Each sub-directory should be named such that it matches the `id` field of the `devcontainer-feature.json`.  Other files can also be included in the feature's sub-directory, and will be included during the [packaging step](#packaging) alongside the two required files.  Any files that are not part of the feature's sub-directory (e.g. outside of `src/dotnet`) will not included in the [packaging step](#packaging).

Optionally, a mirrored `test` directory can be included with an accompanying `test.sh` script.  Implementing tools may use this to run tests against the given feature.

## <a href="#versioning" name="versioning" class="anchor"> Versioning </a>

Each feature is individually [versioned according to the semver specification](https://semver.org/).  The `version` property in the respective `devcontainer-feature.json` file is parsed to determine if the feature should be republished.

Tooling that handles publishing features will not republish features if that exact version has already been published; however, tooling must republish major and minor versions in accordance with the semver specification.

## <a href="#packaging" name="packaging" class="anchor"> Packaging </a>

Features are distributed as tarballs.  The tarball contains the entire contents of the feature sub-directory, including the `devcontainer-feature.json`, `install.sh`, and any other files in the directory.

The tarball is named `devcontainer-feature-<id>.tgz`, where `<id>` is the feature's `id` field.

A reference implementation for packaging and distributing features is provided as a GitHub Action (https://github.com/devcontainers/action).

### <a href="#devcontainer-collection-json" name="devcontainer-collection-json" class="anchor"> devcontainer-collection.json </a>

The `devcontainer-collection.json` is an auto-generated metadata file.

| Property | Type | Description |
| :--- | :--- | :--- |
| sourceInformation | object | Metadata from the implementing packaging tool. |
| features | array | The list of features that are contained in this collection.|
{: .table .table-bordered .table-responsive}

Each features's `devcontainer-feature.json` metadata file is appended into the `features` top-level array.

## <a href="#distribution" name="distribution" class="anchor"> Distribution </a>

There are several supported ways to distribute features.  Distribution is handled by the implementing packaging tool.

A user references a distributed feature in a `devcontainer.json` as defined in ['referencing a feature'](../features#referencing-a-feature).

### <a href="#oci-registry" name="oci-registry" class="anchor"> OCI Registry </a>

An OCI registry that implements the [OCI Artifact Distribution Specification](https://github.com/opencontainers/distribution-spec) serves as the primary distribution mechanism for features.

Each packaged feature is pushed to the registry following the naming convention `<registry>/<namespace>/<id>[:version]`, where version is the major, minor, and patch version of the feature, according to the semver specification.

> The `namespace` is a unique indentifier for the collection of features.  There are no strict rules for the `namespace`; however, one pattern is to set `namespace` equal to source repository's `<owner>/<repo>`. 

A custom media type `application/vnd.devcontainers` and `application/vnd.devcontainers.layer.v1+tar` are used as demonstrated below.

For example, the `go` feature in the `devcontainers/features` namespace at version `1.2.3` would be pushed to the ghcr.io OCI registry.  

_NOTE: The example below uses [`oras`](https://oras.land/) for demonstration purposes.  A supporting tool should directly implement the required functionality from the aforementioned OCI artifact distribution specification._
```bash
# ghcr.io/devcontainers/features/go:1 
REGISTRY=ghcr.io
NAMESPACE=devcontainers/features
FEATURE=go

ARTIFACT_PATH=devcontainer-feature-go.tgz

for VERSION in 1  1.2  1.2.3  latest
do
    oras push ${REGISTRY}/${NAMESPACE}/${FEATURE}:${VERSION} \
            --manifest-config /dev/null:application/vnd.devcontainers \
                             ./${ARTIFACT_PATH}:application/vnd.devcontainers.layer.v1+tar
done
```

`Namespace` is the globally identifiable name for the collection of features. (eg: `owner/repo` for the source code's git repository).

The auto-generated `devcontainer-collection.json` is pushed to the registry with the same `namespace` as above and no accompanying `feature` name. The collection file is always tagged as `latest`.

```bash
# ghcr.io/devcontainers/features
REGISTRY=ghcr.io
NAMESPACE=devcontainers/features

oras push ${REGISTRY}/${NAMESPACE}:latest \
        --manifest-config /dev/null:application/vnd.devcontainers \
                            ./devcontainer-collection.json:application/vnd.devcontainers.collection.layer.v1+json
```

### <a href="#directly-reference-tarball" name="directly-reference-tarball" class="anchor"> Directly Reference Tarball </a>

A feature can be referenced directly in a user's [`devcontainer.json`](../spec#a-hrefdevcontainerjson-namedevcontainerjson-classanchor-devcontainerjson-a) file by HTTPS URI that points to the tarball from the [package step](#packaging).

The `.tgz` archive file must be named `devcontainer-feature-<featureId>.tgz`.

### <a href="#addendum-locally-referenced" name="addendum-locally-referenced" class="anchor"> Addendum: Locally Referenced </a>

To aid in feature authorship, or in instances where a feature should not be published externally, individual features can be referenced locally from the project's file tree.

A local feature is placed in a `.devcontainer/` folder at the root of the [**project workspace folder**](../spec#a-hrefproject-workspace-folder-classanchor-project-workspace-folder-a) and referenced in a user's [`.devcontainer/devcontainer.json`](../spec#a-hrefdevcontainerjson-namedevcontainerjson-classanchor-devcontainerjson-a) by relative path.

The relative path is provided using unix-style path syntax (eg `./<...>`), regardless of the host operating system.

A local feature may **not** be referenced by absolute path, or by a path outside the `.devcontainer/` folder. 

The provided relative path is a path to the folder containing at least a `devcontainer-feature.json` and `install.sh` file, mirroring the structure [previously outlined](#Source-Code).

An example project is illustrated below:

```
.
├── .devcontainer/
│   ├── localFeatureA/
│   │   ├── devcontainer-feature.json
│   │   ├── install.sh
│   │   └── ...
│   ├── localFeatureB/
│   │   ├── devcontainer-feature.json
│   │   ├── install.sh
│   │   └── ...
│   ├── devcontainer.json
```

##### <a href="#devcontainerjson" name="devcontainerjson" class="anchor"> devcontainer.json </a>
```jsonc
{
        // ...
        "features": {
                "./localFeatureA": {},
                "./localFeatureB": {}
        }
}
```
