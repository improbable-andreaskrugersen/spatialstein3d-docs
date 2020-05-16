# Chapter 2 - SpatialOS project setup
## Code branch

https://github.com/improbable-andreaskrugersen/spatialstein3d/tree/chapter2-spatial-setup

## Goals

At the end of this chapter our project will be a SpatialOS project and can be built using the `spatial` CLI. We won't yet use the Worker SDK or start any deployment which will both happen in the [next chapter](chapter3.md).

## Prerequisite reading

These links give you an overview of what SpatialOS is and introduce some important concepts and nomenclature we will use throughout this guide:

https://documentation.improbable.io/spatialos-overview/docs/what-is-spatialos

https://documentation.improbable.io/spatialos-overview/docs/world-entities-components

https://documentation.improbable.io/spatialos-overview/docs/workers-and-load-balancing

https://documentation.improbable.io/spatialos-overview/docs/deployments-and-the-runtime

https://documentation.improbable.io/spatialos-overview/docs/schema

## Structured vs Flexible project layout

SpatialOS offers two flavors of organizing your projects: [Structured Project Layout (SPL)](https://documentation.improbable.io/spatialos-overview/docs/files-and-directories-guide) and [Flexible Project Layout (FPL)](https://documentation.improbable.io/spatialos-overview/docs/flexible-project-layout-introduction). SPL requires your project to have a certain directory structure but is easier to get started with. FPL allows you to structure your project in any way you like. However, FPL is not yet recommended for production use so we will stick with SPL for this guide.

## Make it Spatial

First we need to install the Spatial tools as described in [step 1 here](https://documentation.improbable.io/sdks-and-data/docs/set-up-the-sdks). The Spatial CLI is your main way to build your project, upload assemblies or launch local deployments. We will use this a lot.

Step 2, installing a specific Worker SDK will be handled in the [next chapter](chapter3.md), don't do that yet.

After the `spatial` CLI has been installed, run `spatial help` in your project directory. Among the actual help output, you will also see a warning printed at the top:

```
WARNING: Some commands are only available when you're inside a SpatialOS project directory. Please change your current directory to a SpatialOS project directory to see the additional commands.
```

So, we first need to make our project a SpatialOS project directory. The way to do that is to create a new file at the project root called `spatialos.json`. The format for that file is explained [here](https://documentation.improbable.io/spatialos-overview/docs/spl-project-definition-file). Our version will look like this for now:

> `spatialos.json`
```json
{
    "name": "your_console_project_name",
    "project_version": "1.0.0",
    "sdk_version": "14.6.1"
}
```

You can leave the project name as it is for now. When we make our first cloud deployment we will need to set this properly so that the CLI knows where to upload your assemblies to. For local deployments this name doesn't matter. The latest version of the WorkerSDK is `14.6.1` at the time of this writing.

If you run `spatial help` again, the warning is gone and there are also more sub commands available now, e.g. `worker`.

## Bulding with the Spatial CLI

The next step is to get the project ready to be built via the CLI. The command to do that is `spatial worker build`. If you do that now, you get this output:

```
No workers were found in '/<project path>/spatialstein3d'.
/<project path>/spatialstein3d/schema: error: schema path does not exist.
```

The second one tells us that a mandatory directory required in an SPL project wasn't found. So let's create the `schema` directory. This directory will contain all of our [schema files](https://documentation.improbable.io/sdks-and-data/docs/schema) later. Running `spatial worker build` again now give us this output:

```
No workers were found in '/<project path>/spatialstein3d'.
/tmp/improbable_extracted_packages/bbf2e2637f6ae1761ab7bded18f341e3/schema_compiler: error: No input files.
```

We don't have any schema files yet and since that doesn't make any sense for a Spatial project (after all we want to send some data over network and need to know how that data is going to look like), we need to have some schema files that can be compiled during the build process. We could create a dummy schema file now but instead we are going to load the [standard schema library](https://documentation.improbable.io/sdks-and-data/docs/the-standard-schema-library) provided by Improbable. This library contains a set of schema types and components, some of which are mandatory on all entities, so we are going to need this library in any case.

To get access to the standard schema library, we add a dependency to our `spatialos.json` file:

> `spatialos.json`
```json
{
    "name": "your_console_project_name",
    "project_version": "1.0.0",
    "sdk_version": "14.6.1",
    "dependencies": [
        {
            "name": "standard_library",
            "version": "14.6.1"
        }
    ]
}
```

Running `spatial worker build` again now give us something like this:

```
Transferred  10.0KiB/ 10.0KiB    [====================] 100% [0.0B/s]
Successfully downloaded package type 'schema' with name 'standard_library' and version '14.6.1' to '/tmp/worker-package-download625658379/package.zip'
Extracting packages 1/1    [====================] 100%
No workers were found in '/<project path>/spatialstein3d'.
Generating descriptor. 1/1    [====================] 100%
'spatial worker build' succeeded (2.9s)
```

Our build succeeded! But we haven't actually built anything useful since we don't have any workers yet, which the output kept telling us from the beginning:
```
No workers were found in '/<project path>/spatialstein3d'.
```

Let's fix that.

## Creating a worker

In SPL, workers have to go into the `workers` directory, so let's create that first. You can have separate sub-directories per worker or keep more than one worker in the same directory. In this guide, we'll create a new directory for each worker.

As described on the [worker page](https://documentation.improbable.io/spatialos-overview/docs/workers-and-load-balancing), there are two types of workers: client and server. We will need both going forward but most of the code we have right now is clearly suited to go into a client-worker, so we will create this one first. In a later chapter we start adding a server worker. If you come from a more traditional networking approach, you might find it odd that we start with the client here. After all, in traditional networking, a client is mostly useless if it doesn't have a server to connect to. In SpatialOS both client and server workers connect to a SpatialOS deployment, so it's perfectly reasonable to start with a client to get going! We will, of course, need a server worker to introduce validation, cheat detection etc. but we'll get to that later.

So let's move our existing `client` directory into the `workers` directory.

But we still need to tell SpatialOS more about what our worker is, so we need to create a [worker configuration file](https://documentation.improbable.io/spatialos-overview/docs/spl-worker-configuration-file). As described on the docs page, the file must adhere to a naming convention, so let's name our file `spatialos.client.worker.json`. Our worker's name is `client` now.

Let's start with this file content:

> `workers/client/spatialos.client.worker.json`
```json
{
    "build": {},
    "bridge": {
        "worker_attribute_set": {}
    }
}
```

We still need to specify how to actually build the worker. We can either define all of our build tasks right here under the `build` node or we can keep them in a separate file. Let's go for the latter option for more clarity.

> `workers/client/spatialos.client.worker.json`
```json
{
    "build": {
        "tasks_filename": "build.json"
    },
    "bridge": {
        "worker_attribute_set": {}
    }
}
```

> `workers/client/build.json`
```json
{
    "tasks": [
        {
            "name": "Codegen",
            "steps": [
                {
                    "name": "Dummy"
                }
            ]
        },
        {
            "name": "Build",
            "steps": [
                {
                    "name": "Dummy"
                }
            ]
        },
        {
            "name": "Clean",
            "steps": [
                {
                    "name": "Dummy"
                }
            ]
        }
    ]
}
```

Having defined those three tasks allows us to use the `spatial codegen`, `spatial build` and `spatial clean` commands.

### Code generation

As described on the [build configuration page](https://documentation.improbable.io/spatialos-overview/docs/spl-build-configuration), the `codegen` task should look something like this:

```json
{
    "name": "Codegen",
    "steps": [
        {
            "name": "C++",
            "arguments": [
                "process_schema",
                "generate",
                "--cachePath=../../.spatialos/schema_codegen_cache",
                "--output=../../generated_code/cpp/schema",
                "--language=cpp"
            ]
        }
    ]
}
```

So let's replace our `Dummy` step above with this new content and then run `spatial codegen`. *(Note that we added those dummy steps so that we can actually run `spatial` now. Otherwise the CLI would complain that there are no steps defined for some of the tasks).*

```
Running task 'codegen' for 'client'
[1/1] > Codegen C++
Generating code for cpp. 1/1    [====================] 100%
[1/1] < Codegen C++ (30ms)
succeeded (30ms)
'spatial codegen' succeeded (0.0s)
```

Nice. We now have a new `generated_code` directory in our project root (specified by the `--output=...` parameter in the build step) which contains C++ code for the standard schema library we added as a dependency a short while back. When we add new schema files into the `schema` directory later, their generated code will also be placed here. Note that we don't use any of the generated code yet but we will do so in the following chapters.

Usually you want to run code generation whenever you build your project to make sure that potential changes to schema files are picked up by your code. When we specified `--cachePath=...` in our step, we instructed the schema compiler to keep a cache of compiled schema files in `.schema/schema_codegen_cache`. So it will only compile files that actually changed and it's fast to run. Let's add code generation as our first build step:

```json
{
    "name": "Build",
    "steps": [
        {
            "name": "Codegen",
            "arguments": [
                "invoke-task",
                "Codegen"
            ]
        }
    ]
}
```

`spatial build` gives us:

```
Generating bridge settings for client.
No changes detected, skipping code generation.
Running task 'build' for 'client'
[1/1] > Build Codegen
  [1/1] > Codegen C++
No changes detected, skipping code generation.
  [1/1] < Codegen C++ (10ms)
[1/1] < Build Codegen (10ms)
succeeded (10ms)
'spatial build' succeeded (0.1s)
```

Now we don't need to explicitly run `spatial codegen` but can just run `spatial build`. You can still run `spatial codegen` if you just want to have code generation, of course.

### Build steps

Now let's add a build step to actually build our client!

```json
{
    "name": "Build",
    "steps": [
        {
            "name": "Codegen",
            "arguments": [
                "invoke-task",
                "Codegen"
            ]
        },
        {
            "name": "Client worker",
            "command": "bazel",
            "arguments": [
                "build",
                "//workers/client/src/...",
                "-c",
                "opt"
            ]
        }
    ]
},
```

Run `spatial build` again and you will get something like this:

```
Generating bridge settings for client.
No changes detected, skipping code generation.
Running task 'build' for 'client'
[1/2] > Build Codegen
  [1/1] > Codegen C++
No changes detected, skipping code generation.
  [1/1] < Codegen C++ (10ms)
[1/2] < Build Codegen (10ms)
[2/2] > Build Client worker
<Bazel output here>
[2/2] < Build Client worker (2.42s)
succeeded (2.43s)
'spatial build' succeeded (2.5s)
```

Great, we're almost done for this chapter. Let's also add some steps to our clean task. We need to clean both our generated code as well as our worker binaries:

```json
{
    "name": "Clean",
    "steps": [
        {
            "name": "Generated code",
            "arguments": [
                "process_schema",
                "clean",
                "--cachePath=../../.spatialos/schema_codegen_cache",
                "../../.spatialos/schema_codegen_proto",
                "../../generated_code/cpp/schema"
            ]
        },
        {
            "name": "Workers",
            "command": "bazel",
            "arguments": [
                "clean"
            ]
        }
    ]
}
```

Running `spatial clean` should give something like this:

```
Running task 'clean' for 'client'
[1/2] > Clean Generated code
Cleaning the following directories [../../.spatialos/schema_codegen_cache ../../.spatialos/schema_codegen_proto ../../generated_code/cpp/schema].
[1/2] < Clean Generated code (< 10ms)
[2/2] > Clean Workers
INFO: Starting clean (this may take a while). Consider using --async if the clean takes more than several minutes.
[2/2] < Clean Workers (80ms)
succeeded (80ms)
'spatial clean' succeeded (0.1s)
```

## Cleaning up

`spatial` adds some new directories to our project directory, so we should add these lines to our `.gitignore`:

```
.spatialos/
build/
generated_code/*/
logs/
```

Note: we will add a `BUILD` file to the `generated_code` folder later, that's why we are explicitly only ignoring sub-folders here.

## Conclusion

And that's it for now! To build, just run `spatial build`. To run the client, we still have to do `bazel-bin/workers/client/src/spatialstein3d` (note the path has changed slightly). This will change next chapter, when we will actually start using the Worker SDK and connect our client worker to a local deployment.
