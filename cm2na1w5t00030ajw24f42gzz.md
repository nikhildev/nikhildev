---
title: "Simple hello world program using Bazel and Go lang"
datePublished: Tue Oct 15 2024 21:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cm2na1w5t00030ajw24f42gzz
slug: simple-hello-world-program-using-bazel-and-go-lang
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1729772757533/22499f0c-7700-4a2b-93d6-d1829eaa253f.jpeg
tags: bazel, go

---

After writing the post on [Building with Go in a monorepo using Bazel, Gazelle and bzlmod](https://nixclix.com/building-with-go-in-a-monorepo-using-bazel-gazelle-and-bzlmod/) and sharing with some colleagues, I saw some growing interest in monorepo development. I learnt that not many people were still experienced enough to understand the benefits it could bring. So I decided to convert this to a series starting with this post.

In this post, I will try to cover some extremely basic concepts of Bazel along with code snippets to get someone up and going in no time.

**What is Bazel?**

According to the [official documentation](https://bazel.build/about/intro?ref=nixclix.com)

> Bazel is an open-source build and test tool similar to Make, Maven, and Gradle. It uses a human-readable, high-level build language. Bazel supports projects in multiple languages and builds outputs for multiple platforms. Bazel supports large codebases across multiple repositories, and large numbers of users.

It has been in use at Google for decades and is thoroughly battle tested. You can read more about how I became aware of this on the post linked above.

---

**Step 1: Setting up repository**

For the purpose of this series I have created a repository at [*https://github.com/nixclix/basil*](https://github.com/nixclix/basil?ref=nixclix.com) which will evolve over time. At the time of writing this post, the latest commit is [*https://github.com/nixclix/basil/tree/d8af2aea6fb8b692f735f9959429df9fcd28422b*](https://github.com/nixclix/basil/tree/d8af2aea6fb8b692f735f9959429df9fcd28422b?ref=nixclix.com)

So, go ahead an create a new git repo on any provider you want to choose

For the `.gitignore` I highly recommend adding the following so as to not include unnecessary files in your commits

```plaintext
# If you prefer the allow list template instead of the deny list, see community template:
# https://github.com/github/gitignore/blob/main/community/Golang/Go.AllowList.gitignore
#
bazel-*
/.ijwb
/.clwb
/.idea
/.project
*.swp
/.bazelrc.user

# macOS-specific excludes
.DS_Store

# Binaries for programs and plugins
*.exe
*.exe~
*.dll
*.so
*.dylib

# Test binary, built with `go test -c`
*.test

# Output of the go coverage tool, specifically when used with LiteIDE
*.out

# Dependency directories (remove the comment below to include it)
# vendor/

# Go workspace file
go.work
go.work.sum

# env file
.env
```

---

**Step 2: Adding some Bazel boilerplate**

Starting Bazel 6.3 you do not need to have a [WORKSPACE file anymore](https://bazel.build/external/migration?ref=nixclix.com). So, we will start with creating the following at the root of the repo

`MODULE.bazel`

```plaintext
"""Building go applications in a monorepo environment"""

module(name = "basil", version = "0.0.0")
http_file = use_repo_rule(
    "@bazel_tools//tools/build_defs/repo:http.bzl", "http_file"
)
http_archive = use_repo_rule(
    "@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive"
)

# https://github.com/bazelbuild/rules_go/blob/master/docs/go/core/bzlmod.md
bazel_dep(name = "rules_go", version = "0.50.1")
bazel_dep(name = "gazelle", version = "0.39.1")

GO_VERSION = "1.23.2"

go_sdk = use_extension("@rules_go//go:extensions.bzl", "go_sdk")
go_sdk.download(version = GO_VERSION)

go_deps = use_extension("@gazelle//:extensions.bzl", "go_deps")
go_deps.from_file(go_mod = "//:go.mod")
```

Here is where we set the go version we want to use as well as the gazelle and rules\_go version.

We will be using *Gazelle* for BUILD file management. Gazelle makes generating build file quite convenient. You can read more about it [here](https://github.com/bazel-contrib/bazel-gazelle?ref=nixclix.com)

`BUILD.bazel`

```plaintext
load("@gazelle//:def.bzl", "gazelle")

gazelle(name = "gazelle")

gazelle(
    name = "gazelle-update-repos",
    args = [
        "-from_file=go.mod",
        "-to_macro=deps.bzl%go_dependencies",
        "-prune",
    ],
    command = "update-repos",
)
```

This should be at the root of the repo. This instructs gazelle on the source of the go mod file and other processes to run

Next we create 3 more files with the respective contents. Don't worry about what these do for now.

.bazelignore

```text
runfiles/
bzlmod/
```

.bazelrc

```plaintext
# Enable Bzlmod for every Bazel command
common --enable_bzlmod
```

.bazelversion

```plaintext
7.3.1
```

Alright, by this point you should have everything that's needed to get some basics working. Now if you run `bazel build //...` at the root, bazel should be able traverse through the repo and build any packages that it discovers. Bazel caches package outputs from earlier builds, so any subsequent build from here on should be extremely fast.

Next up, let's get cracking on that actual hello world program.

---

**Step 3: Writing a helloworld package**

For the basic organisation of code we will be writing all the Go code in a directory called `/packages`. This way any references in other parts of the code can refer to it as `//packages/...`

Let's create a directory in the packages directory called `helloworld` and add the following

`helloworld.go`

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello World!")
}
```

`BUILD.bazel`

```plaintext
load("@rules_go//go:def.bzl", "go_binary", "go_library")

go_binary(
    name = "helloworld",
    embed = [":helloworld_lib"],
    importpath = "basil/packages/helloworld",
    visibility = ["//visibility:private"],
)

go_library(
    name = "helloworld_lib",
    srcs = ["helloworld.go"],
    importpath = "basil/packages/helloworld_lib",
    visibility = ["//visibility:private"],
)
```

The go program is fairly simple. We have a simple main function that simple prints a hello world message.

The bit we are interested in is the `BUILD.bazel` file. Let's go over this block by block and try to understand what it means.

The first line loads the `go_binary` and `go_library` macros from `rules_go` . You will find these being used later in the code.

In line#10 we are defining a library called `helloworld_lib` and specify the sources of the library as `helloworld.go`. If this package needs to be me made importable, then you can also specify the path it will be available at which is `basil/packages/helloworld_lib`. Then comes the visibility and here we are specifying that the library is private only visible within this package. In the future posts we might look into how to change these parameters to use dependencies from other packages

In line#3 we then use the `go_binary` macro from rules\_go to generate a binary for our program. Here we specify the library that we have defined earlier in the `embed` parameter. The package visibility applies to binaries as well.

---

**Step 4: Build and Run**

And that's it. Voila! You can run the binary by first building it using `bazel build //packages/helloworld` followed by `bazel run //packages/helloworld`

If you like this post and would like to get future posts that build upon this as a part of the series, please subscribe.