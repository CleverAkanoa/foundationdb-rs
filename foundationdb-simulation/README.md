# foundationdb-simulation

This crate provides the tools to write Rust workloads that can be loaded and executed in the
official FoundationDB simulator, allowing for rigorous and deterministic testing of Rust layers.

## How It Works

FoundationDB's simulation framework includes an `ExternalWorkload` that can dynamically load a
shared object at runtime. This shared object must expose specific symbols that the simulator calls
to drive the workload.

Originally, `ExternalWorkload` used a C++ interface. This is cumbersome because the C++ ABI is not
stable and most languages do not interoperate with it easily. As of FoundationDB 7.4, the
`ExternalWorkload` supports a pure C interface, which this crate targets. For backwards
compatibility with FoundationDB 7.1 and 7.3, a C++ shim can be compiled to translate the C
interface back to the C++ one.

### For FoundationDB 7.4 and Newer (Recommended)

- Uses a pure C API (FFI-safe).
- **No need** to build inside the official FoundationDB Docker image.
- Requires setting `useCAPI=true` in the test configuration file.
- Results in faster build times and a simpler setup.

### For FoundationDB 7.1 and 7.3

- Requires a C++ shim to bridge the C and C++ ABIs.
- **Must** be built within the [official `foundationdb/build` Docker image](https://hub.docker.com/r/foundationdb/build)).
- The linker must be set to `clang`.
- Involves a more complex build process due to C++ ABI compatibility requirements.

## Setup

First, create a new Rust project with a library structure:

```console
.
├── Cargo.toml
└── src/
    └── lib.rs
```

Next, add `foundationdb-simulation` to your `Cargo.toml` and configure your crate as `cdylib`
as the `ExternalWorkload` expects a shared object.

```toml
[lib]
name = "myworkload"
crate-type = ["cdylib"]

[dependencies]
# Make sure to select the feature flag for your target FDB version.
foundationdb-simulation = { version = "...", features = ["fdb-7_4"] } # or "fdb-7_1", "fdb-7_3"
```

## Compilation

If you are targeting FoundationDB 7.1 or 7.3 (which require the C++ shim), you **must** compile
your workload inside the official FoundationDB Docker image. Compiling outside of this environment
will almost certainly lead to segmentation faults at runtime due to C++ ABI mismatches.

Compile your workload using the standard Cargo commands. This will create a shared object in
`./target/release/` (`./target/debug/`). The filename will be `lib<name>.so`, where `<name>` is
the `name` you set in the `[lib]` section of your `Cargo.toml`. For example: `libmyworkload.so`.

## Launching the Simulation

The FoundationDB simulator is launched via the `fdbserver` binary with a TOML configuration file.

```bash
fdbserver -r simulation -f ./test_file.toml
```

Your test file must specify `testName=External` to use the `ExternalWorkload` framework.

```toml
testTitle = "MyRustTest"
testName = "External"

# The name passed to your workload factory
workloadName = "MyWorkload" 

# The path to the directory containing your .so file
libraryPath = "./target/release/" 
# The name of your library from Cargo.toml (without lib/.so)
libraryName = "myworkload" 

# Required for FDB 7.4+ when using the C API
useCAPI = true

# Custom options for your workload
myCustomOption = 42
```

## Workload Definition

FoundationDB workloads are defined by implementing the `RustWorkload` trait:

```rust
pub trait RustWorkload {
    async fn setup(&mut self, db: SimDatabase);
    async fn start(&mut self, db: SimDatabase);
    async fn check(&mut self, db: SimDatabase);
    fn get_metrics(&self, out: Metrics);
    fn get_check_timeout(&self) -> f64;
}
```

The simulator expects the shared object to expose a factory that can create workload instances.
This crate provides two traits to define these factories.

For simple use cases where a single workload implementation is defined, you can implement the
`SingleRustWorkload` trait directly on your `RustWorkload`.

```rust
pub trait SingleRustWorkload: RustWorkload {
    const FDB_API_VERSION: u32;
    fn new(name: String, context: WorkloadContext) -> Self;
}
```

Then, register your workload with the corresponding `register_workload` macro.

For more complex scenarios, where multiple workload implementations exist in a single shared
object, you can define a separate factory that implements the `RustWorkloadFactory` trait. This
allows selecting the implementation based on the workload name provided in the test configuration.

```rust
pub trait RustWorkloadFactory {
    const FDB_API_VERSION: u32;
    fn create(name: String, context: WorkloadContext) -> WrappedWorkload;
}
```

The factory must be registered with the corresponding `register_factory` macro.

> **Warning:** Do not use more than one "register macro" per project.

See the `atomic` and `noop` implementations in the `examples/` directory for complete working examples.

## The WorkloadContext

The `WorkloadContext` passed at instantiation is the primary way to interact with the simulator.
It provides several useful methods:

- `trace(severity, name, details)`: Add a log entry to the FDB trace files.
- `now()`: Get the current simulated time as a `f64`.
- `rnd()`: Get a deterministic 32-bit random number.
- `shared_random_number()`: Get a deterministic 64-bit random number (the same for all clients).
- `client_id()`: Get the ID of the current client.
- `client_count()`: Get the total number of clients in the simulation.
- `get_option<T>(name)`: Get a custom configuration option from the test file.

### Get options
In the simulation configuration file you can add custom parameters to your workload.
These parameters can be read with `WorkloadContext::get_option`. This method will first try to get
the parameter value as a raw string and then convert it in a the type of your choice.
If the parameter doesn't exist, its value is invalid or set to null, the function returns None.

Example:

```rust
impl SingleRustWorkload for MyWorkload {
    fn new(name: String, context: WorkloadContext) -> Self {
        let my_custom_option: usize = context
            .get_option("myCustomOption")
            .unwrap();
        Self { context, name, my_custom_option }
    }
}
```

### Tracing

Use `WorkloadContext::trace` to log messages with a given severity and a map of string "details".
A severity of `Severity::Error` will automatically stop the `fdbserver` process.

```rust
impl RustWorkload for MyWorkload {
    fn setup(&mut self, db: SimDatabase) {
        self.context.trace(
            Severity::Info,
            "SuccessfullySetupWorkload",
            details![
                "Layer" => "Rust",
                "Phase" => "setup",
                "Name" => self.name,
                "CustomOption" => self.my_custom_option,
            ],
        );
    }
    ...
}
```

### Randomness

To maintain determinism, all random numbers must be sourced from the simulator.
Use `WorkloadContext::rnd()` or `WorkloadContext::shared_random_number()` for a shared seed.
Do not use external entropy sources like `rand::thread_rng()`.

# Workload Lifecycle

## Instantiation

Once the `fdbserver` process is ready, it loads your shared object and calls your workload factory
to create instances. The simulator creates a random number of "clients," and your factory will be
called once for each client.

> **Note:** Contrary to the C++ `ExternalWorkload` which has separate `create` and `init` methods,
> the `RustWorkload` is not created until the simulator's `init` phase.

The `WorkloadContext` passed to your factory should be stored in your `RustWorkload`, as it is safe
to use across phases and will not be provided again.

## The setup, start, and check Phases

These three phases run sequentially. The simulation will not begin the `start` phase until all
clients have completed the `setup` phase, and so on.

Each function is `async` and runs cooperatively alonside the `fdbserver`.
As long as your function is executing, the `fdbserver` is paused to ensure determinism.
You **must** yield control back to the simulator for any database operations to occur.
Do not "busy wait" for a foundationdb function to return as it will deadlock.

FoundationDB Simulation runs your async code on a custom future executor that integrates with
the `fdbserver` event loop. It effectively turns your async fn into a sequence of cooperative
steps, allowing the `fdbserver` to drive simulation events between awaits.

- All foundationdb-rs async operations are compatible with this model.
- Avoid using arbitrary async primitives from other crates (e.g., `tokio::sleep`).

## Reporting Metrics

At the end of the simulation, the `get_metrics` method is called for each client.
Implement this method to report results.

```rust
fn get_metrics(&self, out: Metrics) {
    // val metrics are summed across all clients
    out.push(Metric::val("total_ops", self.ops as f64));
    // avg metrics are averaged across all clients
    out.push(Metric::avg("avg_latency", self.latency / self.ops.max(1) as f64));
}
```

# Common Mistakes

* **Compiling C++ shim outside of the Docker.** C++ ABI is extremely environment-dependent, not
  compiling in the exact same environment as the other half of the interface will probably result
  in segmentation faults or mangled strings that will trigger unrecoverable errors.

* **Forgetting to read all options.** Any custom option in the configuration that is not read will
  trigger a `Workload had invalid options.` error.

* **Forgetting useCAPI=true.** If the `fdbserver` used supports the C API, you still need to add
  explicitely this option, otherwise the `ExternalWorkload` will try to load the C++ symbol.

* **Using pointers or references after a phase ends.** After each phase, the simulator may move or
  deallocate memory. Any pointers or references to FoundationDB objects become invalid. You must
  use the fresh `&mut self` and `db` references passed to the next phase. Storing `SimDatabase`,
  `Transaction`, or `Future` objects across phases will lead to segmentation faults or other
  undefined behavior.
