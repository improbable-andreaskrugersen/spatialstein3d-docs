# Chapter 3 - Connecting to a deployment
## Code branch

https://github.com/improbable-andreaskrugersen/spatialstein3d/tree/chapter3-client-deployment

## Goals

Last chapter we created a basic SpatialOS project setup and we can now build our project using `spatial build`. In this chapter we will start using the Worker SDK to connect to a deployment and turn our code into a real client-worker.

## Choice of Worker SDK language

Since the project was written in C++ so far, we have two choices which Worker SDK to use:
- [Worker SDK in C++](https://documentation.improbable.io/sdks-and-data/docs/cpp-introduction)
- [Worker SDK in C](https://documentation.improbable.io/sdks-and-data/docs/c-introduction)

The C++ SDK is more geared towards evaluation than to be used in a production game but it has the advantage of code generation for our schema files. Since this project is not supposed to be a production-ready game and we want to get up and running quickly, we'll use the C++ API. Feel free to read the introduction pages linked above to get more information about the trade-offs of using either one.

## Get the worker packages

Let's complete our [C++ setup](https://documentation.improbable.io/sdks-and-data/docs/cpp-setup-and-installation) in this chapter. First we need to get access to the Worker SDK. For that we need to download the matching worker packages for your system using the spatial package service. While we could do that by calling `spatial package get` manually, a more consistent way is to create a [worker packages file](https://documentation.improbable.io/spatialos-overview/docs/spl-worker-packages-file) and specify the packages we need there. The file must be located in the worker's directory and must be named `spatialos_worker_packages.json`.

We need two different packages. Since the Worker SDK in C++ is just a wrapper around the Worker SDK in C, we need:
- The Worker SDK in C
- The C++ header package

If we were using the Worker SDK in C in this project, we would be getting the C headers package instead. You can find a list of the relevant packages for all supported platforms [here](https://documentation.improbable.io/sdks-and-data/docs/c-setup-and-installation#section-packages).

> `workers/client/spatialos_worker_packages.json`
```json
{
    "targets": [
        {
            "path": "../../dependencies/worker_sdk/headers",
            "type": "worker_sdk",
            "packages": [
                {
                    "name": "cpp_headers"
                }
            ]
        },
        {
            "path": "../../dependencies/worker_sdk/linux/lib",
            "type": "worker_sdk",
            "packages": [
                {
                    "name": "c-static-x86_64-gcc510_pic-linux",
                    "platform": "linux"
                }
            ]
        }
    ]
}
```

> TODO: Windows

Now we still need to trigger downloading the packages. We'll do this as a separate build step in our `workers/client/build.json`, right after the `Codegen` step:

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
            "name": "Install dependencies",
            "arguments": [
                "package",
                "unpack"
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

Running `spatial build` again now downloads our requested packages:

```
[2/3] > Build Install dependencies
Transferred 426.2KiB/426.2KiB    [====================] 100% [4.1MiB/s]
Successfully downloaded package type 'worker_sdk' with name 'cpp_headers' and version '14.6.1' to '/tmp/worker-package-download232806432/package.zip'
Extracting packages 1/1    [====================] 100%
Transferred  35.2MiB/ 35.2MiB    [====================] 100% [4.1MiB/s]
Successfully downloaded package type 'worker_sdk' with name 'c-static-x86_64-gcc510_pic-linux' and version '14.6.1' to '/tmp/worker-package-download924381951/package.zip'
Extracting packages 1/1    [====================] 100%
[2/3] < Build Install dependencies (13.55s)
```

In our worker packages file we specified where to put the extracted packages. They are now located in our `dependencies/worker_sdk` directory in our project root.

Worker packages are cached locally to avoid subsequent downloads. The location depends on your platform, on Linux they are located in `~/.improbable/cache/worker_package`.

> TODO: Windows

While we're here, let's also add a clean step:

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
            "name": "Dependencies",
            "arguments": [
                "package",
                "clean"
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

This will now additionally remove the extracted packages from our project directory when we run `spatial clean`.

## Add the Worker SDK to our build process

Now we need to make sure that the extracted worker packages are actually used when building our worker. This is specific to our chosen buildsystem Bazel and we won't discuss the details here. Have a look at [dependencies/worker_sdk/BUILD](https://github.com/improbable-andreaskrugersen/spatialstein3d/blob/chapter3-client-deployment/dependencies/worker_sdk/BUILD) to see the code for creating a library target for the Worker SDK. Finally, we need to add this target as a dependency to our `workers/client/src/BUILD` file:

> TODO: verify link

```
SHARED_DEPS = [
    "//dependencies/eigen:eigen",
    "//dependencies/worker_sdk:worker_sdk",
]
```

Let's also update our `.gitignore` and ignore the downloaded SDK files:

```
dependencies/worker_sdk/*/
```

## Add a connection to our client worker

Now let's add some actual code to establish a connection with a deployment. As described on [Connect to SpatialOS](https://documentation.improbable.io/sdks-and-data/docs/cpp-connect-to-spatialos), we have two choices: Whether to run against a local deployment or a cloud deployment. If we were to release our game, we would run against a cloud deployment, so that all our players could reach it. But we are not yet at this stage, so let's go with a local deployment for now. It's also faster to start and requires less setup.

Additionally, there are two ways to connect to a deployment: Using the Receptionist or the Locator. Since we're building a client which is going to run outside of the actual cloud deployment, we will have to use the Locator flow here at some point. This requires a bit more setup to work locally though. So let's keep our setup to a minimum for now and use the Receptionist flow for a local deployment. So our configuration is to use `worker::Connection` directly with `UseExternalIp == false`.

In the spirit of minimal setup, we'll add our connection in a quick & dirty way to our `main()` function. We'll clean this up in the next chapter. For now we just want to make sure we can connect to the local deployment.

```cpp
#include <improbable/worker.h>

using AllComponents = worker::Components<>;

// ...

int main() {
// ...
    AllComponents allComponents{};

    worker::LogsinkParameters logsink;
    logsink.Type = worker::LogsinkType::kStdout;
    logsink.FilterParameters.Categories = worker::LogCategory::kApi;
    logsink.FilterParameters.Level = worker::LogLevel::kInfo;

    worker::ConnectionParameters params;
    params.WorkerType = "client";
    params.Network.ConnectionType = worker::NetworkConnectionType::kModularKcp;
    params.Network.UseExternalIp = false;
    params.Logsinks = {logsink};
    params.EnableLoggingAtStartup = true;

    worker::Connection connection =
        worker::Connection::ConnectAsync(allComponents, "localhost", 7777, "client", params)
            .Get();
    if (connection.GetConnectionStatusCode() == worker::ConnectionStatusCode::kSuccess) {
        std::cout << "Connection successful" << std::endl;
    }
// ...
```

What's going on here? First we need to include the Worker SDK. `<improbable/worker.h>` is the only file you need to include, it includes all other header files of the Worker SDK. Next we need to [provide the components](https://documentation.improbable.io/sdks-and-data/docs/cpp-provide-components) we are going to use in our worker, which we call `AllComponents`. We are not using any components yet, so we leave the template parameters empty for now. *(Note: If we were using FPL and invoked the schema compiler directly, we could let it auto-generate this type for us)*

> TODO: Link to schema bundle generation

Next we set up a log sink to get some log output from the Worker SDK. See [this page](https://documentation.improbable.io/sdks-and-data/docs/debug-and-troubleshoot-workers) for more information. Basically we're saying "Give me high-level logs (`kApi`) of level `kInfo` or higher and pipe them to `stdout`". Feel free to, e.g., use `LogCategory::kAll` here if you'd like to see more of what's going on under the hood.

We need to configure our connection. There's a whole variety of parameters to choose from and we're not going to talk about them here. We use the minimum required setup (+ adding logging). Just note that we're using the `ModularKCP` protocol here which is [the recommended protocol](https://documentation.improbable.io/spatialos-overview/docs/network-configuration#section-choosing-a-network-stack) for client workers. Finally, we establish a connection to the Receptionist. `7777` is the default port for the Receptionist running in a local deployment. You can choose any worker ID you like, we just went with `client` here. `worker::Connection::ConnectAsync` returns a `Future`. If your game doesn't want to block during connecting, you can check the future periodically to see if the connection has been established. For now we just block until we have a connection.

Now if we were to build and start our client again the way we've done it before with `bazel-bin/workers/client/src/spatialstein3d`, we'd get a bunch of log messages, including this one:

```
[Error] (API) Failed to connect to the receptionist: gRPC error INTERNAL: error connecting to localhost:7777: failed to connect TCP socket: hang up during connection
```

This is to be expected, of course. After all, we haven't started a local deployment yet, so let's do that now!

## Setting up a local deployment

There are four things we need to do for our initial setup of a local deployment:
- Create an assembly
- Add a launch configuration
- Configure our worker for a local deployment
- Connect the client to the deployment

### Create an assembly

For cloud deployments we need to upload an [assembly](https://documentation.improbable.io/sdks-and-data/docs/glossary#section-assembly) which is basically just a zip file that's extracted to a remote directory. For local deployments we can do the same or run the executable directly. In our case it's better to use the zip approach since we need to bundle our binary with assets and we'll need it later anyway.

So let's add another build step that creates a zip file of our binary and assets. In this case the step will just execute a shell script that zips up all the files we need. Feel free to change this approach for your own projects. It might also be worthwhile to look into the `spatial file zip` command.

> `workers/client/build.json`
```
{
    "name": "Build",
    "steps": [
        ... previous steps ...
        {
            "name": "Client worker",
            "command": "bazel",
            "arguments": [
                "build",
                "//workers/client/src/...",
                "-c",
                "opt"
            ]
        },
        {
            "name": "Assembly",
            "command": "./make_zip.sh"
        }
    ]
},
```

`make_zip.sh` follows the same naming convention as `spatial file zip` and creates a zip file called `client@Linux.zip` and puts it into `build/assembly/worker`. This directory is the expected location for all your built assemblies in SPL.

> TODO: Windows


### Add a launch configuration

Now we need to set up a [launch configuration file](https://documentation.improbable.io/spatialos-overview/docs/spl-launch-configuration-file). The launch configuration basically defines world parameters and can also specify which workers to start automatically. We can create multiple files to have different launch configurations. In our case we just need a single one so we can make use of the naming convention `default_launch.json` and we won't need to explicitly specify the configuration we want to use when we start the local deployment.

> `default_launch.json`
```json
{
    "template": "w2_r0500_e5",
    "world": {
        "chunk_edge_length_meters": 50,
        "snapshots": {
            "snapshot_write_period_seconds": 0
        },
        "dimensions": {
            "x_meters": 50,
            "z_meters": 50
        }
    }
}
```

> TODO: explain content

Now run `spatial local launch` and you will get an output like this:

```
Preparing to run SpatialOS.
No changes detected, skipping code generation.
'spatial prepare-for-run' succeeded (0.0s)
SpatialOS will not launch from a snapshot since the default snapshot file '/<project path>/spatialstein3d/snapshots/default.snapshot' could not be found. Please create a snapshot before starting SpatialOS.
SpatialOS starting.
Downloading new SpatialOS Runtime version 14.5.2...
Transferred  67.4MiB/ 67.4MiB    [====================] 100% [4.0MiB/s]
Extracted SpatialOS Runtime to /tmp/fabric_bundles/14.5.2.
[improbable.worker.assembly.WorkerAssemblyProviderFactory] Loaded worker assemblies for these workers types: client.
[improbable.module.ModuleNodeHelpers] Using the new Runtime including new bridge, load balancer, and entity database.
[improbable.worker.assembly.WorkerAssemblyProvider] The following worker configurations do not contain valid managed launch configurations for the current platform LINUX, and the runtime will be unable to launch managed workers of this type: [client]. This is expected for worker types you intend to only connect manually.
[improbable.loadbalancing.v2.config.LoadbalancerV2Config] Inferred the following new-format loadbalancing strategy from the deployment configuration:

[improbable.module.ModuleNode] SpatialOS runtime startup completed in 1.436s.
SpatialOS ready. (20.1s)
Access the Inspector at http://localhost:21000/inspector
Access the new Inspector at http://localhost:21000/inspector-v2
```

At this point our local deployment is running! But we don't have any workers connected to it yet.

### Configure the worker for a local deployment

As described on [Deploy locally](https://documentation.improbable.io/sdks-and-data/docs/deploy-locally), we now need to manually connect our client worker to the deployment. The previous output when starting the deployment also indicated that our client worker can only be connected manually since it doesn't have a managed launch configuration:

```
[improbable.worker.assembly.WorkerAssemblyProvider] The following worker configurations do not contain valid managed launch configurations for the current platform LINUX, and the runtime will be unable to launch managed workers of this type: [client]. This is expected for worker types you intend to only connect manually.
```

This is perfectly fine for our use case, we don't want to start clients automatically when the deployment starts. So let's configure what `spatial` should do when we ask it to connect our worker manually. We need to create an external launch configuration in our worker configuration file:

> `workers/client/spatialos.client.worker.json`
```json
{
    "build": {
        "tasks_filename": "build.json"
    },
    "bridge": {
        "worker_attribute_set": {
            "attributes": ["client"]
        }
    },
    "external": {
        "local": {
            "run_type": "EXECUTABLE_ZIP",
            "linux": {
                "artifact_name": "client@Linux.zip",
                "command": "./spatialstein3d",
                "arguments": []
            }
        }
    }
}
```

The existence of the `external` section says that this is a worker that can connect to the deployment manually. `local` is the default configuration name for a local deployment. You can choose any name here, this will be needed in the next step. The `run_type` says that we are using a zip file as an assembly. If we wanted to run an executable directly (only for local deployments) we could say "EXECUTABLE" here. The remaining fields specify the assembly name to use and which binary inside that archive to execute after it has been extracted. We could also specify command line arguments and we will make use of this feature later.

### Connect our client worker to the deployment

That's all the setup we need. Make sure your local deployment is running at this point, otherwise run `spatial local launch`. Now run `spatial local worker launch client local` and behold our client running as a client worker in our local deployment!

You should see this output:

```
[Info ] (API) <Connection> created with use_external_ip: false, connection_type: ModularKcp, multiplex_level: 1, security_type: D(TLS), send_queue_capacity: 4096, receive_queue_capacity: 4096
[Warn ] (API) No components found. It is likely that the required generated code has not been properly included.
Connection successful
```

Ignore the warning about no components for now. We'll fix that in the next chapter when we start creating entities and components.

Let's quickly dig into the command we used: `spatial local worker launch` basically says "launch a new worker and connect it to the local deployment". `client` is the worker type (remember we named our worker configuration file `spatialos.client.worker.json`. If we had chosen a different name instead of `client` there, we'd need to use this name instead). The final `local` is the name of the external configuration to use. If your worker uses command line parameters, for example, you could have different launch configurations in your worker configuration file and then start the one with the correct parameters here.

## Conclusion

In this chapter we started a local deployment and set up our client worker to connect to that deployment using the receptionist flow. The worker doesn't do anything useful with that connection yet but we now have completed the bulk of the boiler plate. Next chapter we'll go and create some entities and components.

To run, you now need to terminals and run these commands:
- Local deployment: `spatial build && spatial local launch`
- Client worker: `spatial local worker launch client local`
