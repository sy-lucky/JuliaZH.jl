# Pkg

## 介绍

Pkg 是 Julia 1.0 及更新版本的标准包管理器。与那些安装和管理单个全局软件包集的传统包管理器不同，Pkg 是围绕「环境」设计的：独立的软件包集可以是单个项目的本地软件包集，也可按名称共享和选择。 环境中包及其版本的确切信息保存在_清单文件_中，该文件可以检入项目存储库并在版本控制中进行跟踪，从而显着提高项目的可重复性。如果你曾经尝试运行未曾使用过的代码，但发现其完全无法工作，而这只是因为你更新或卸载了一些你的项目所使用的包，那么你会理解这种方法的意图。在 Pkg 中，由于每个项目都维护着各自的独立软件包集，你再也不会遇到此问题了。 此外，如果你签出项目到新系统中，你可以简单地搭建出其清单文件所描述的环境，并立即在具有已知的良好依赖项集的环境下启动并运行该项目。

由于环境是彼此独立地进行安装和管理的，在 Pkg 中显著缓解了「依赖地狱」。你如果想在新项目中使用更新和更好的包，但在不同的项目中使用旧版本包，那也没问题——因为它们的环境是彼此分离的，所以它们可以使用不同版本的包，两个版本的包都同时安装在系统的不同的位置中。每个包版本的位置都是规范的，所以当多个环境使用包的相同版本时，它们可以共享已安装的包，从而避免不必要的包重复。包管理器会定期「垃圾收集」不再被任何环境使用的旧版本包。

Pkg 对本地环境的处理方法可能让曾经使用过 Python 的 `virtualenv` 或 Ruby 的 `bundler` 的人感到熟悉。在 Julia 中，我们不仅没有通过破解语言的代码加载机制来支持环境，而且还有 Julia 本身就理解它们的好处。此外，Julia 环境是「可堆叠的」：你可以将一个环境叠加在另一个环境上，从而可以访问主环境之外的其它包。这使得更容易在提供主环境的项目上工作，同时依然访问所有你常用的开发工具，如分析器、调试器等，这只需在加载路径中更后地包含具有这些开发环境的路径。

Last but not least, Pkg is designed to support federated package registries.
This means that it allows multiple registries managed by different parties to
interact seamlessly. In particular, this includes private registries which can
live behind corporate firewalls. You can install and update your own packages
from a private registry with exactly the same tools and workflows that you use
to install and manage official Julia packages. If you urgently need to apply a
hotfix for a public package that’s critical to your company’s product, you can
tag a private version of it in your company’s internal registry and get a fix to
your developers and ops teams quickly and easily without having to wait for an
upstream patch to be accepted and published. Once an official fix is published,
however, you can just upgrade your dependencies and you'll be back on an
official release again.

## 词汇表

**项目（Project）：**一个具有标准布局的源代码树，包括了用来放置主要的 Julia 代码的 `src` 目录、用来放置测试的 `test` 目录、用来放置文档的 `docs` 目录和可选的用来放置构建脚本及其输出的 `deps` 目录。项目通常有一个项目文件和一个可选的清单文件：

- **项目文件（Project file）：**一个在项目根目录下的文件，叫做 `Project.toml`（或 `JuliaProject.toml`），用来描述项目的元数据，包括项目的名称、UUID（针对包）、作者、许可证和它所依赖的包和库的名称及 UUID。
   
   
   

- **清单文件（Manifest file）：**一个在项目根目录下的文件，叫做 `Manifest.toml`（或 `JuliaManifest.toml`），用来描述完整的依赖关系图、每个包的确切版本以及项目使用的库。
   
   

**包（Package）：**一个提供可重用功能的项目，其它 Julia 项目可以同 `import X` 或 `using X` 使用它。一个包应该包含一个具有 `uuid` 条目（此条目给出该包 UUID）的项目文件。此 UUID 用于在依赖它的项目中标识该包。

!!! note
    由于遗留原因，可以在 REPL 或脚本的顶级中加载没有项目文件或 UUID 的包。但是，无法在具有项目文件或 UUID 的项目中加载没有它们的包。一旦你曾从项目文件加载包，所有包就都需要项目文件和 UUID。

**应用（application）：**一个提供独立功能的项目，不打算被其它 Julia 项目重用。例如，Web 应用、命令行工具或者科学论文附带的模拟或分析代码。应用可以有 UUID 但也可以没有。应用还可以为其所依赖的包提供全局配置选项。另一方面，包不可能提供全局配置，因为这可能与主应用的配置相冲突。

!!! note
    **项目 _vs._ 包 _vs._ 应用：**

    1. **项目**是一个总称：包和应用都是一种项目。
    2. **包**应该有 UUID，而应用可以有也可以没有。
    3. **应用**可以提供全局的配置，而包不行。

**Library (future work):** a compiled binary dependency (not written in Julia)
packaged to be used by a Julia project. These are currently typically built in-
place by a `deps/build.jl` script in a project’s source tree, but in the future
we plan to make libraries first-class entities directly installed and upgraded
by the package manager.

**环境（Environment）：**项目文件和清单文件的组合，项目文件与依赖关系图相结合后提供了顶级名称映射，而清单文件提供了包到它们入口点的映射。有关的详细信息，请参阅手册中代码加载的相关章节。

- **Explicit environment:** an environment in the form of an explicit project
   
   
   

- **Implicit environment:** an environment provided as a directory (without a
   
   
   
   
   
   
   

**Registry:** a source tree with a standard layout recording metadata about a
registered set of packages, the tagged versions of them which are available, and
which versions of packages are compatible or incompatible with each other. A
registry is indexed by package name and UUID, and has a directory for each
registered package providing the following metadata about it:

- name——例如 `DataFrames`
- UUID——例如 `a93c6f00-e57d-5684-b7b6-d8193f3e46c0`
- authors——例如 `Jane Q. Developer <jane@example.com>`
- license——例如 MIT，BSD3 或 GPLv2
- repository——例如 `https://github.com/JuliaData/DataFrames.jl.git`
- description——一个总结包功能的文本块
- keywords——例如 `data`，`tabular`，`analysis`，`statistics`
- versions——所有已注册版本的标签列表

每个包的已注册版本都会提供以下信息：

- its semantic version number – e.g. `v1.2.3`
- its git tree SHA-1 hash – e.g. `7ffb18ea3245ef98e368b02b81e8a86543a11103`
- a map from names to UUIDs of dependencies
- which versions of other packages it is compatible/incompatible with

Dependencies and compatibility are stored in a compressed but human-readable
format using ranges of package versions.

**Depot:** a directory on a system where various package-related resources live,
including:

- `environments`: shared named environments (e.g. `v0.7`, `devtools`)
- `clones`: bare clones of package repositories
- `compiled`: cached compiled package images (`.ji` files)
- `config`: global configuration files (e.g. `startup.jl`)
- `dev`: default directory for package development
- `logs`: log files (e.g. `manifest_usage.toml`, `repl_history.jl`)
- `packages`: installed package versions
- `registries`: clones of registries (e.g. `General`)

**Load path:** a stack of environments where package identities, their
dependencies, and entry-points are searched for. The load path is controlled in
Julia by the `LOAD_PATH` global variable which is populated at startup based on
the value of the `JULIA_LOAD_PATH` environment variable. The first entry is your
primary environment, often the current project, while later entries provide
additional packages one may want to use from the REPL or top-level scripts.

**Depot path:** a stack of depot locations where the package manager, as well as
Julia's code loading mechanisms, look for registries, installed packages, named
environments, repo clones, cached compiled package images, and configuration
files. The depot path is controlled by the Julia `DEPOT_PATH` global variable
which is populated at startup based on the value of the `JULIA_DEPOT_PATH`
environment variable. The first entry is the “user depot” and should be writable
by and owned by the current user. The user depot is where: registries are
cloned, new package versions are installed, named environments are created and
updated, package repos are cloned, newly compiled package image files are saved,
log files are written, development packages are checked out by default, and
global configuration data is saved. Later entries in the depot path are treated
as read-only and are appropriate for registries, packages, etc. installed and
managed by system administrators.

## 入门

在 Julia REPL 中使用 `]` 键即可进入 Pkg 模式。

```
(v0.7) pkg>
```

提示符括号内的部分显示当前项目的名称。由于我们尚未创建自己的项目，我们正处于默认项目中，其位于 `~/.julia/environments/v0.7`（或任何你恰巧在运行的 Julia 版本）。

要返回 `julia>` 提示符，请在输入行为空时按退格键或直接按 Ctrl+C。可通过调用 `pkg>help` 获得帮助。如果你所处的环境无法访问 PEPL，你仍可以通过字符串宏 `pkg`（其在 `using Pkg` 后可用）使用 REPL 模式的命令。命令 `pkg"cms"` 将等价于在 RPEL 模式中执行 `cmd`。

此处的文档介绍了如何使用 REPL 的 Pkg 模式。使用 Pkg API（通过调用 `Pkg.` 函数）的文档正在编写中。

### 添加包

有两种方法可以添加包，分别是使用 `add` 命令和 `dev` 命令。最常用的是 `add`，我们首先介绍它的用法。

#### 添加已注册的包

在 REPL 的 Pkg 模式中，添加包可以使用 `add` 命令，其后接包的名称，例如：

```
(v0.7) pkg> add Example
   Cloning default registries into /Users/kristoffer/.julia/registries
   Cloning registry General from "https://github.com/JuliaRegistries/General.git"
  Updating registry at `~/.julia/registries/General`
  Updating git-repo `https://github.com/JuliaRegistries/General.git`
 Resolving package versions...
  Updating `~/.julia/environments/v0.7/Project.toml`
  [7876af07] + Example v0.5.1
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [7876af07] + Example v0.5.1
  [8dfed614] + Test
```

Here we added the package Example to the current project. In this example, we are using a fresh Julia installation,
and this is our first time adding a package using Pkg. By default, Pkg clones Julia's General registry,
and uses this registry to look up packages requested for inclusion in the current environment.
The status update shows a short form of the package UUID to the left, then the package name, and the version.
Since standard libraries (e.g. `Test`) are shipped with Julia, they do not have a version. The project status contains the packages
you have added yourself, in this case, `Example`:

```
(v0.7) pkg> st
    Status `Project.toml`
  [7876af07] Example v0.5.1
```

The manifest status, in addition, includes the dependencies of explicitly added packages.

```
(v0.7) pkg> st --manifest
    Status `Manifest.toml`
  [7876af07] Example v0.5.1
  [8dfed614] Test
```

It is possible to add multiple packages in one command as `pkg> add A B C`.

After a package is added to the project, it can be loaded in Julia:

```
julia> using Example

julia> Example.hello("User")
"Hello, User"
```

A specific version can be installed by appending a version after a `@` symbol, e.g. `@v0.4`, to the package name:

```
(v0.7) pkg> add Example@0.4
 Resolving package versions...
  Updating `~/.julia/environments/v0.7/Project.toml`
  [7876af07] + Example v0.4.1
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [7876af07] + Example v0.4.1
```

If the master branch (or a certain commit SHA) of `Example` has a hotfix that has not yet included in a registered version,
we can explicitly track a branch (or commit) by appending `#branch` (or `#commit`) to the package name:

```
(v0.7) pkg> add Example#master
  Updating git-repo `https://github.com/JuliaLang/Example.jl.git`
 Resolving package versions...
  Updating `~/.julia/environments/v0.7/Project.toml`
  [7876af07] ~ Example v0.5.1 ⇒ v0.5.1+ #master (https://github.com/JuliaLang/Example.jl.git)
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [7876af07] ~ Example v0.5.1 ⇒ v0.5.1+ #master (https://github.com/JuliaLang/Example.jl.git)
```

The status output now shows that we are tracking the `master` branch of `Example`.
When updating packages, we will pull updates from that branch.

To go back to tracking the registry version of `Example`, the command `free` is used:

```
(v0.7) pkg> free Example
 Resolving package versions...
  Updating `~/.julia/environments/v0.7/Project.toml`
  [7876af07] ~ Example v0.5.1+ #master (https://github.com/JuliaLang/Example.jl.git) ⇒ v0.5.1
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [7876af07] ~ Example v0.5.1+ #master )https://github.com/JuliaLang/Example.jl.git) ⇒ v0.5.1
```


#### Adding unregistered packages

If a package is not in a registry, it can still be added by instead of the package name giving the URL to the repository to `add`.

```
(v0.7) pkg> add https://github.com/fredrikekre/ImportMacros.jl
  Updating git-repo `https://github.com/fredrikekre/ImportMacros.jl`
 Resolving package versions...
Downloaded MacroTools ─ v0.4.1
  Updating `~/.julia/environments/v0.7/Project.toml`
  [e6797606] + ImportMacros v0.0.0 # (https://github.com/fredrikekre/ImportMacros.jl)
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [e6797606] + ImportMacros v0.0.0 # (https://github.com/fredrikekre/ImportMacros.jl)
  [1914dd2f] + MacroTools v0.4.1
```

The dependencies of the unregistered package (here `MacroTools`) got installed.
For unregistered packages we could have given a branch (or commit SHA) to track using `#`, just like for registered packages.


#### Adding a local package

Instead of giving a URL of a git repo to `add` we could instead have given a local path to a git repo.
This works similarly to adding a URL. The local repository will be tracked (at some branch) and updates
from that local repo are pulled when packages are updated.
Note that changes to files in the local package repository will not immediately be reflected when loading that package.
The changes would have to be committed and the packages updated in order to pull in the changes.

#### Developing packages

By only using `add` your Manifest will always have a "reproducible state", in other words, as long as the repositories and registries used are still accessible
it is possible to retrieve the exact state of all the dependencies in the project. This has the advantage that you can send your project (`Project.toml`
and `Manifest.toml`) to someone else and they can "instantiate" that project in the same state as you had it locally.
However, when you are developing a package, it is more convenient to load packages at their current state at some path. For this reason, the `dev` command exists.

Let's try to `dev` a registered package:

```
(v0.7) pkg> dev Example
  Updating git-repo `https://github.com/JuliaLang/Example.jl.git`
 Resolving package versions...
  Updating `~/.julia/environments/v0.7/Project.toml`
  [7876af07] + Example v0.5.1+ [`~/.julia/dev/Example`]
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [7876af07] + Example v0.5.1+ [`~/.julia/dev/Example`]
```

The `dev` command fetches a full clone of the package to `~/.julia/dev/` (the path can be changed by setting the environment variable `JULIA_PKG_DEVDIR`).
When importing `Example` julia will now import it from `~/.julia/dev/Example` and whatever local changes have been made to the files in that path are consequently
reflected in the code loaded. When we used `add` we said that we tracked the package repository, we here say that we track the path itself.
Note that the package manager will never touch any of the files at a tracked path. It is therefore up to you to pull updates, change branches etc.
If we try to `dev` a package at some branch that already exists at `~/.julia/dev/` the package manager we will simply use the existing path.
For example:

```
(v0.7) pkg> dev Example
  Updating git-repo `https://github.com/JuliaLang/Example.jl.git`
[ Info: Path `/Users/kristoffer/.julia/dev/Example` exists and looks like the correct package, using existing path instead of cloning
```

Note the info message saying that it is using the existing path. As a general rule, the package manager will
never touch files that are tracking a path.

If `dev` is used on a local path, that path to that package is recorded and used when loading that package.
The path will be recorded relative to the project file, unless it is given as an absolute path.

To stop tracking a path and use the registered version again, use `free`

```
(v0.7) pkg> free Example
 Resolving package versions...
  Updating `~/.julia/environments/v0.7/Project.toml`
  [7876af07] ↓ Example v0.5.1+ [`~/.julia/dev/Example`] ⇒ v0.5.1
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [7876af07] ↓ Example v0.5.1+ [`~/.julia/dev/Example`] ⇒ v0.5.1
```

It should be pointed out that by using `dev` your project is now inherently stateful.
Its state depends on the current content of the files at the path and the manifest cannot be "instantiated" by someone else without
knowing the exact content of all the packages that are tracking a path.

Note that if you add a dependency to a package that tracks a local path, the Manifest (which contains the whole dependency graph) will become
out of sync with the actual dependency graph. This means that the package will not be able to load that dependency since it is not recorded
in the Manifest. To update sync the Manifest, use the REPL command `resolve`.

### Removing packages

Packages can be removed from the current project by using `pkg> rm Package`.
This will only remove packages that exist in the project, to remove a package that only
exists as a dependency use `pkg> rm --manifest DepPackage`.
Note that this will remove all packages that depends on `DepPackage`.

### Updating packages

When new versions of packages the project is using are released, it is a good idea to update. Simply calling `up` will try to update *all* the dependencies of the project
to the latest compatible version. Sometimes this is not what you want. You can specify a subset of the dependencies to upgrade by giving them as arguments to `up`, e.g:

```
(v0.7) pkg> up Example
```

The version of all other packages direct dependencies will stay the same. If you only want to update the minor version of packages, to reduce the risk that your project breaks, you can give the `--minor` flag, e.g:

```
(v0.7) pkg> up --minor Example
```

Packages that track a repository are not updated when a minor upgrade is done.
Packages that track a path are never touched by the package manager.

### Pinning a package

A pinned package will never be updated. A package can be pinned using `pin` as for example

```
(v0.7) pkg> pin Example
 Resolving package versions...
  Updating `~/.julia/environments/v0.7/Project.toml`
  [7876af07] ~ Example v0.5.1 ⇒ v0.5.1 ⚲
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [7876af07] ~ Example v0.5.1 ⇒ v0.5.1 ⚲
```

Note the pin symbol `⚲` showing that the package is pinned. Removing the pin is done using `free`

```
(v0.7) pkg> free Example
  Updating `~/.julia/environments/v0.7/Project.toml`
  [7876af07] ~ Example v0.5.1 ⚲ ⇒ v0.5.1
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [7876af07] ~ Example v0.5.1 ⚲ ⇒ v0.5.1
```

### Testing packages

The tests for a package can be run using `test`command:

```
(v0.7) pkg> test Example
   Testing Example
   Testing Example tests passed
```

### Building packages

The build step of a package is automatically run when a package is first installed.
The output of the build process is directed to a file.
To explicitly run the build step for a package the `build` command is used:

```
(v0.7) pkg> build MbedTLS
  Building MbedTLS → `~/.julia/packages/MbedTLS/h1Vu/deps/build.log`

shell> cat ~/.julia/packages/MbedTLS/h1Vu/deps/build.log
┌ Warning: `wait(t::Task)` is deprecated, use `fetch(t)` instead.
│   caller = macro expansion at OutputCollector.jl:63 [inlined]
└ @ Core OutputCollector.jl:63
...
[ Info: using prebuilt binaries
```

## Creating your own projects

So far we have added packages to the default project at `~/.julia/environments/v0.7`, it is, however, easy to create other, independent, projects.
It should be pointed out if two projects uses the same package at the same version, the content of this package is not duplicated.
In order to create a new project, create a directory for it and then activate that directory to make it the "active project" which package operations manipulate:

```
shell> mkdir MyProject

shell> cd MyProject
/Users/kristoffer/MyProject

(v0.7) pkg> activate .

(MyProject) pkg> st
    Status `Project.toml`
```

Note that the REPL prompt changed when the new project is activated. Since this is a newly created project, the status command show it contains no packages, and in fact, it has no project or manifest file until we add a package to it:

```
shell> ls -l
total 0

(MyProject) pkg> add Example
  Updating registry at `~/.julia/registries/General`
  Updating git-repo `https://github.com/JuliaRegistries/General.git`
 Resolving package versions...
  Updating `Project.toml`
  [7876af07] + Example v0.5.1
  Updating `Manifest.toml`
  [7876af07] + Example v0.5.1
  [8dfed614] + Test

shell> ls -l
total 8
-rw-r--r-- 1 stefan staff 207 Jul  3 16:35 Manifest.toml
-rw-r--r-- 1 stefan staff  56 Jul  3 16:35 Project.toml

shell> cat Project.toml
[deps]
Example = "7876af07-990d-54b4-ab0e-23690620f79a"

shell> cat Manifest.toml
[[Example]]
deps = ["Test"]
git-tree-sha1 = "8eb7b4d4ca487caade9ba3e85932e28ce6d6e1f8"
uuid = "7876af07-990d-54b4-ab0e-23690620f79a"
version = "0.5.1"

[[Test]]
uuid = "8dfed614-e22c-5e08-85e1-65c5234f0b40"
```

This new environment is completely separate from the one we used earlier.

## Garbage collecting old, unused packages

As packages are updated and projects are deleted, installed packages that were once used will inevitably
become old and not used from any existing project.
Pkg keeps a log of all projects used so it can go through the log and see exactly which projects still exist
and what packages those projects used. The rest can be deleted.
This is done with the `gc` command:

```
(v0.7) pkg> gc
    Active manifests at:
        `/Users/kristoffer/BinaryProvider/Manifest.toml`
        ...
        `/Users/kristoffer/Compat.jl/Manifest.toml`
   Deleted /Users/kristoffer/.julia/packages/BenchmarkTools/1cAj: 146.302 KiB
   Deleted /Users/kristoffer/.julia/packages/Cassette/BXVB: 795.557 KiB
   ...
   Deleted /Users/kristoffer/.julia/packages/WeakRefStrings/YrK6: 27.328 KiB
   Deleted 36 package installations: 113.205 MiB
```

Note that only packages in `~/.julia/packages` are deleted.

## Creating your own packages

A package is a project with a `name`, `uuid` and `version` entry in the `Project.toml` file `src/PackageName.jl` file that defines the module `PackageName`.
This file is executed when the package is loaded.

### Generating files for a package

To generate files for a new package, use `pkg> generate`.

```
(v0.7) pkg> generate HelloWorld
```

This creates a new project `HelloWorld` with the following files (visualized with the external [`tree` command](https://linux.die.net/man/1/tree)):

```jl
shell> cd HelloWorld

shell> tree .
.
├── Project.toml
└── src
    └── HelloWorld.jl

1 directory, 2 files
```

The `Project.toml` file contains the name of the package, its unique UUID, its version, the author and eventual dependencies:

```toml
name = "HelloWorld"
uuid = "b4cd1eb8-1e24-11e8-3319-93036a3eb9f3"
version = "0.1.0"
author = ["Some One <someone@email.com>"]

[deps]
```

The content of `src/HelloWorld.jl` is:

```jl
module HelloWorld

greet() = print("Hello World!")

end # module
```

We can now activate the project and load the package:

```jl
pkg> activate .

julia> import HelloWorld

julia> HelloWorld.greet()
Hello World!
```

### Adding dependencies to the project

Let’s say we want to use the standard library package `Random` and the registered package `JSON` in our project.
We simply `add` these packages (note how the prompt now shows the name of the newly generated project,
since we are inside the `HelloWorld` project directory):

```
(HelloWorld) pkg> add Random JSON
 Resolving package versions...
  Updating "~/Documents/HelloWorld/Project.toml"
 [682c06a0] + JSON v0.17.1
 [9a3f8284] + Random
  Updating "~/Documents/HelloWorld/Manifest.toml"
 [34da2185] + Compat v0.57.0
 [682c06a0] + JSON v0.17.1
 [4d1e1d77] + Nullables v0.0.4
 ...
```

Both `Random` and `JSON` got added to the project’s `Project.toml` file, and the resulting dependencies got added to the `Manifest.toml` file.
The resolver has installed each package with the highest possible version, while still respecting the compatibility that each package enforce on its dependencies.

We can now use both `Random` and `JSON` in our project. Changing `src/HelloWorld.jl` to

```
module HelloWorld

import Random
import JSON

greet() = print("Hello World!")
greet_alien() = print("Hello ", Random.randstring(8))

end # module
```

and reloading the package, the new `greet_alien` function that uses `Random` can be used:

```
julia> HelloWorld.greet_alien()
Hello aT157rHV
```

### Adding a build step to the package.

The build step is executed the first time a package is installed or when explicitly invoked with `build`.
A package is built by executing the file `deps/build.jl`.

```
shell> cat deps/build.log
I am being built...

(HelloWorld) pkg> build
  Building HelloWorld → `deps/build.log`
 Resolving package versions...

shell> cat deps/build.log
I am being built...
```

If the build step fails, the output of the build step is printed to the console

```
shell> cat deps/build.jl
error("Ooops")

(HelloWorld) pkg> build
  Building HelloWorld → `deps/build.log`
 Resolving package versions...
┌ Error: Error building `HelloWorld`:
│ ERROR: LoadError: Ooops
│ Stacktrace:
│  [1] error(::String) at ./error.jl:33
│  [2] top-level scope at none:0
│  [3] include at ./boot.jl:317 [inlined]
│  [4] include_relative(::Module, ::String) at ./loading.jl:1071
│  [5] include(::Module, ::String) at ./sysimg.jl:29
│  [6] include(::String) at ./client.jl:393
│  [7] top-level scope at none:0
│ in expression starting at /Users/kristoffer/.julia/dev/Pkg/HelloWorld/deps/build.jl:1
└ @ Pkg.Operations Operations.jl:938
```

### Adding tests to the package

When a package is tested the file `test/runtests.jl` is executed.

```
shell> cat test/runtests.jl
println("Testing...")
(HelloWorld) pkg> test
   Testing HelloWorld
 Resolving package versions...
Testing...
   Testing HelloWorld tests passed
```

#### Test-specific dependencies

Sometimes one might want to use some packages only at testing time but not
enforce a dependency on them when the package is used. This is possible by
adding dependencies to `[extras]` and a `test` target in `[targets]` to the Project file.
Here we add the `Test` standard library as a test-only dependency by adding the
following to the Project file:

```
[extras]
Test = "8dfed614-e22c-5e08-85e1-65c5234f0b40"

[targets]
test = ["Test"]
```

We can now use `Test` in the test script and we can see that it gets installed on testing:

```
shell> cat test/runtests.jl
using Test
@test 1 == 1

(HelloWorld) pkg> test
   Testing HelloWorld
 Resolving package versions...
  Updating `/var/folders/64/76tk_g152sg6c6t0b4nkn1vw0000gn/T/tmpPzUPPw/Project.toml`
  [d8327f2a] + HelloWorld v0.1.0 [`~/.julia/dev/Pkg/HelloWorld`]
  [8dfed614] + Test
  Updating `/var/folders/64/76tk_g152sg6c6t0b4nkn1vw0000gn/T/tmpPzUPPw/Manifest.toml`
  [d8327f2a] + HelloWorld v0.1.0 [`~/.julia/dev/Pkg/HelloWorld`]
   Testing HelloWorld tests passed```
```

### Compatibility

Compatibility refers to the ability to restrict what version of the dependencies that your project is compatible with.
If the compatibility for a dependency is not given, the project is assumed to be compatible with all versions of that dependency.

Compatibility for a dependency is entered in the `Project.toml` file as for example:

```toml
[compat]
Example = "0.4.3"
```

After a compatibility entry is put into the project file, `up` can be used to apply it.

The format of the version specifier is described in detail below.

!!! info
    There is currently no way to give compatibility from the Pkg REPL mode so for now, one has to manually edit the project file.

#### Version specifier format

Similar to other package managers, the Julia package manager respects [semantic versioning](https://semver.org/) (semver).
As an example, a version specifier is given as e.g. `1.2.3` is therefore assumed to be compatible with the versions `[1.2.3 - 2.0.0)` where `)` is a non-inclusive upper bound.
More specifically, a version specifier is either given as a **caret specifier**, e.g. `^1.2.3`  or a **tilde specifier** `~1.2.3`.
Caret specifiers are the default and hence `1.2.3 == ^1.2.3`. The difference between a caret and tilde is described in the next section.
The intersection of multiple version specifiers can be formed by comma separating indiviual version specifiers.

##### Caret specifiers

A caret specifier allows upgrade that would be compatible according to semver.
An updated dependency is considered compatible if the new version does not modify the left-most non zero digit in the version specifier.

Some examples are shown below.

```
^1.2.3 = [1.2.3, 2.0.0)
^1.2 = [1.2.0, 2.0.0)
^1 =  [1.0.0, 2.0.0)
^0.2.3 = [0.2.3, 0.3.0)
^0.0.3 = [0.0.3, 0.0.4)
^0.0 = [0.0.0, 0.1.0)
^0 = [0.0.0, 1.0.0)
```

While the semver specification says that all versions with a major version of 0 are incompatible with each other, we have made that choice that
a version given as `0.a.b` is considered compatible with `0.a.c` if `a != 0` and  `c >= b`.

##### Tilde specifiers

A tilde specifier provides more limited upgrade possibilities. With a tilde, only the last specified digit is allowed to increment by one.
This gives the following example.

```
~1.2.3 = [1.2.3, 1.2.4)
~1.2 = [1.2.0, 1.3.0)
~1 = [1.0.0, 2.0.0)
```

#### Inequality specifiers

Inequalities can also be used to specify version ranges:

```
>= 1.2.3 = [1.2.3,  ∞)
≥ 1.2.3 = [1.2.3,  ∞)
= 1.2.3 = [1.2.3, 1.2.3]
< 1.2.3 = [0.0.0, 1.2.2]
```


## Precompiling a project

The REPL command `precompile` can be used to precompile all the dependencies in the project. You can for example do

```
(HelloWorld) pkg> update; precompile
```

to update the dependencies and then precompile them.

## Preview mode

If you just want to see the effects of running a command, but not change your state you can `preview` a command.
For example:

```
(HelloWorld) pkg> preview add Plots
```

or

```
(HelloWorld) pkg> preview up
```

will show you the effects of adding `Plots`, or doing a full upgrade, respectively, would have on your project.
However, nothing would be installed and your `Project.toml` and `Manifest.toml` are untouched.

## Using someone else's project

Simply clone their project using e.g. `git clone`, `cd` to the project directory and call

```
(v0.7) pkg> activate .

(SomeProject) pkg> instantiate
```

If the project contains a manifest, this will install the packages in the same state that is given by that manifest.
Otherwise, it will resolve the latest versions of the dependencies compatible with the project.

## References

This section describes the "API mode" of interacting with Pkg.jl which is recommended for non-interactive usage,
in i.e. scripts. In the REPL mode packages (with associated version, UUID, URL etc) are parsed from strings,
for example, `"Package#master"`,`"Package@v0.1"`, `"www.mypkg.com/MyPkg#my/feature"`.
It is possible to use strings as arguments for simple commands in the API mode (like `Pkg.add(["PackageA", "PackageB"])`,
more complicated commands, that e.g. specify URLs or version range, uses a more structured format over strings.
This is done by creating an instance of a [`PackageSpec`](@ref) which are passed in to functions.

```@docs
PackageSpec
PackageMode
UpgradeLevel
Pkg.add
Pkg.develop
Pkg.activate
Pkg.rm
Pkg.update
Pkg.test
Pkg.build
Pkg.pin
Pkg.free
Pkg.instantiate
Pkg.resolve
Pkg.setprotocol!
```
