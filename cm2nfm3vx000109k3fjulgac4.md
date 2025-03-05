---
title: "Building with Go in a monorepo using Bazel, Gazelle and bzlmod"
datePublished: Thu Sep 26 2024 21:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cm2nfm3vx000109k3fjulgac4
slug: building-with-go-in-a-monorepo-using-bazel-gazelle-and-bzlmod
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1729782054482/72c88c26-d9d6-41c8-a30a-d0e6b0c6a8d5.jpeg
tags: bazel, go, monorepo

---

This post is relevant to any engineer who joined in a small-ish company and stuck around long enough to watch it grow, scale, hire more engineers, form more specialised teams. These are good problems to deal with. By the time I quit Google in 2016, it had mastered the infrastructure of being able to support a very large number of engineering population being able to derive benefits from everything historic to still have the capability to grow perhaps 10x or more. One part of the art that Google had mastered is developing in a monorepo. Google internally uses a repo called "google3" which is where all code (at least the one I was aware of resided). This was the mother of everything tech that Google runs. Right from the ability to commit code, getting reviews, cross team/functions/org dependencies, etc. All this was possible by this amazing build system that Google developed called "blaze" and in March 2015 they made it accessible to everyone outside Google under the name "bazel" which is just an anagram for blaze. So, why all this psycho-babble you ask?

When I joined [Wolt](https://wolt.com/?ref=nixclix.com) in 2020 we still called ourselves a startup who still had full autonomy into how, where and what software we wrong. This meant that we had the freedom to pick the language of our choice, the framework, build system and how we deployed. As engineers we had to deal with a lot of boiler plate by ourself over and over, all the way from managing infrastructure to how the binaries are built. At times we spent more time setting things up for a new service than writing the actual useful code.

Then as we started scaling up came the painful aspect of working with a massive crowd.

1. How do we share common code between teams?
    
2. How do we ensure that service although distributed are testable in one go?
    
3. How do we orchestrate releases of feature where there are more than one services involved?
    
4. How do we reduce the friction for engineers to work on cross ownership code?
    
5. Any many more
    

So the question is how do we build and environment where adopt all the good aspects of a monorepo but also retain the freedom, safety and speed that microservices provide.

In this post I will try to do my best to document who some of my relearning and hopefully provide a scaffold for anyone else who might be interested.

The following are the versions this post will be working with.

```go
bazel v7.3.1
gazelle v0.39.0
rules_go v0.50.1
go v1.23.1
```

This is purely for the purpose of the latest with best compatibility. Please feel free to comment if I might be mistaken

There are some simple simple goals we will try to achieve here

1. Code is well structured despite being beyond team boundaries
    
2. Reusability is a key. Anything that could be helpful to other teams should be placed in a common places and relevant definition need to be there to make it usable like a library
    
3. You should be able to run any part of the monorepo locally without much knowledge of setting up SDKs, compilers, etc.
    

This post will not cover how to deploy the binaries.

---

The name of the repo that I have created for the sake of this post os called"basil". You can call it whatever you want.

**Step 1**: Starting Bazel 6.3 you do not need to have a [WORKSPACE file anymore](https://bazel.build/external/migration?ref=nixclix.com). So, we will start with creating the following at the root of the repo

*MODULE.bazel*

```go
module(name = "basil", version = "0.0.0")
http_file = use_repo_rule(
    "@bazel_tools//tools/build_defs/repo:http.bzl", "http_file"
)
http_archive = use_repo_rule(
    "@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive"
)

# https://github.com/bazelbuild/rules_go/blob/master/docs/go/core/bzlmod.md
bazel_dep(name = "rules_go", version = "0.41.0")
bazel_dep(name = "gazelle", version = "0.37.0")

GO_VERSION = "1.23.1"

go_sdk = use_extension("@rules_go//go:extensions.bzl", "go_sdk")
go_sdk.download(version = GO_VERSION)

go_deps = use_extension("@gazelle//:extensions.bzl", "go_deps")
go_deps.from_file(go_mod = "//:go.mod")

use_repo(go_deps, "com_github_google_go_cmp")
```

The module file does the following

1. Define the report for [rules\_go](https://github.com/bazelbuild/rules_go?ref=nixclix.com) and [gazelle](https://github.com/bazelbuild/bazel-gazelle?ref=nixclix.com)
    
2. Define the version of go that we will be using for the SDK and the gazelle extensions for the same
    

**Step 2:** Then some more boilerplate in the BUILD file

*BUILD*

```go
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

This also ensure that dependencies from go.mod from the role are provided

Now that this is out of way we need to decide what kind of a code organisation we would like to have. For the sake of this post we will assume that all of the code will be in a directory called `packages` , we have two team who work adjacently called `ledgering` and `invoicing` who can share code between themselves as well as from common `libs` package. This sort of hierarchy is fairly common and extensible.

**Step 3:** So let's create a directory called `packages` Next, run the following at the root

```shell
bazel run //:gazelle
```

Gazelle is a build file generator for bazel. Writing these files with dependencies, naming, etc can get quite cumbersome. Gazelle can inspect your code and add the necessary library dependencies it the build file.

**Step 4:** Next, in the root of the monorepo, run the following to initialise the `go.mod` file

```shell
go mod init basil
go mody tidy
```

---

**Step 5:** Alright, now we are ready to write to actual code. I will start by creating the following directories in the `packages` directory

1. `libs` to hold all common libraries which could be nested if needed as well
    
2. `invoicing` for the first team which can serve as a provider for invoices
    
3. `ledgering` for the second team that can serve as a consumer for invoices
    

**Step 6:** Let's start with the common libs directory which can potential hold some formatting functions. Let's create another directory called `common` in libs.

**Step 7:** Next in the formatters directory create a file called `invoice_id_generator.go` and place the following code in that

```go
package libs

import (
	"fmt"
	"math/rand"
)

func GetInvoiceId() string {
	return fmt.Sprint(int64(rand.Int()))
}

```

Now when you run `bazel run //:gazelle` on the root, you can see that Gazelle creates a build for you automatically.

```go
load("@rules_go//go:def.bzl", "go_library")

go_library(
    name = "formatters",
    srcs = ["concatinator.go"],
    importpath = "basil/packages/libs/formatters",
    visibility = ["//visibility:public"],
)
```

Gazelle by default creates a `go_library` but we will see what else we can create manually later

**Step 8:** Now that we have something useful let's imagine that ledgering team is requesting the invoicing team to create an invoices which in turn uses a common library from libs to generate the id.

We start by creating a file called `invoice.go` in the team's root with the following content

```go
package invoicer

import (
	"math/rand"
	"time"
)

type Invoice struct {
	ID        int64
	Amount    float64
	CreatedAt time.Time
}

func GetNewInvoice() *Invoice {
	return &Invoice{
		ID:        int64(rand.Int()),
		Amount:    rand.Float64(),
		CreatedAt: time.Now(),
	}
}
```

And then another file called `booker.go` in ledgering with the following contents

```go
package main

import (
	lib "basil/packages/fintech/invoicing"
	"fmt"
)

func main() {
	i := lib.GetNewInvoice()
	fmt.Println("Generate new Invoice with ID:", i.ID)
}
```

Now when you run `blaze run //:gazelle` you will notice that gazelle has create the libaries along with the dependencies that the go code uses as well.

**Step 9:** Now, we want the make the `booker.go` an executable since it might be invoked as a standalone. Since gazelle has create a binary for us, let's create one with the following contents in the BUILD.bazel. So, your final build file should look like

```go
load("@rules_go//go:def.bzl", "go_binary", "go_library")

go_binary(
    name = "app",
    embed = [":lib"],
    importpath = "basil/packages/fintech/ledgering_app",
    visibility = ["//visibility:public"],
)

go_library(
    name = "lib",
    srcs = ["booker.go"],
    importpath = "basil/packages/fintech/ledgering",
    visibility = ["//visibility:private"],
    deps = ["//packages/fintech/invoicing:lib"],
)
```

**Step 10:** Now its time to take this whole thing for a spin. You can build and run the binary as shows below

```shell
bazel build //packages/ledgering:app
bazel run //packages/ledgering:app
```

If all went fine, you should see something like below

```go
INFO: Analyzed target //packages/ledgering:app (1 packages loaded, 5 targets configured).
INFO: Found 1 target...
Target //packages/ledgering:app up-to-date:
  bazel-bin/packages/ledgering/app_/app
INFO: Elapsed time: 0.747s, Critical Path: 0.52s
INFO: 4 processes: 1 internal, 3 darwin-sandbox.
INFO: Build completed successfully, 4 total actions
INFO: Running command line: bazel-bin/packages/ledgering/app_/app
Generate new Invoice with ID: 7213007727604049262
```

---

And there you are. You no longer have to create individual repositories for each project that you might be working on but instead use same repo and libraries across projects. I will probably write a follow up post to this on how you can build binaries through CI/CD for this setup.

Please do leave comments in case I have mispoken or misunderstand anything. You can find all the code I have demoed above in my repo