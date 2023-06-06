# tokio-bin-process

[![Crates.io](https://img.shields.io/crates/v/tokio-bin-process.svg)](https://crates.io/crates/tokio-bin-process)
[![Docs](https://docs.rs/tokio-bin-process/badge.svg)](https://docs.rs/tokio-bin-process)
[![dependency status](https://deps.rs/repo/github/shotover/tokio-bin-process/status.svg)](https://deps.rs/repo/github/shotover/tokio-bin-process)

Allows your integration tests to run your application under a separate process and assert on tokio-tracing events.

To achieve this, it builds your application as a native binary,
runs it with tokio-tracing in json mode,
and then processes the json to both assert on the logs and display the logs in human readable form.

It is a little opinionated and by default will fail the test when a tracing warning or error occurs.
However a specific warning or error can be allowed on a per test basis.

## Usage

Example usage for an imaginary database project named cooldb:

```rust
/// you'll want a helper like this as you'll be creating this in every integration test.
async fn cooldb_process() -> BinProcess {
    // start the process
    let mut process = BinProcess::start_with_args(
        "cooldb", // The name of the crate in this workspace to run
        "cooldb", // The name that BinProcess should prepend its forwarded logs with
        &[
            // provide any custom CLI args required
            "--foo", "bar",
            // tokio-bin-process relies on reading tracing json's output,
            // so configure the application to produce that.
            "--log-format", "json"
        ],
        None
    )
    .await;

    // block asynchrounously until the application gives an event indicating that its ready
    tokio::time::timeout(
        Duration::from_secs(30),
        process.wait_for(
            &EventMatcher::new()
                .with_level(Level::Info)
                .with_target("cooldb")
                .with_message("accepting inbound connections"),
        ),
    )
    .await
    .unwrap();
    process
}

#[tokio::test]
async fn test_some_functionality() {
    // start the db
    let cooldb = cooldb_process().await;

    // connect to the db, do something and assert we get the expected result
    perform_test();

    // Shutdown the DB, asserting that no warnings or errors occured,
    // but allow and expect a certain warning.
    // A drop bomb ensures that the test will fail if we forget to call this method.
    cooldb
        .shutdown_and_then_consume_events(&[
            EventMatcher::new()
                .with_level(Level::Warn)
                .with_target("cooldb::internal")
                .with_message("The user did something silly that we want to warn about but is actually expected in this test case")
        ])
        .await;
}
```

## Application side setup

There is a non-trival amount of setup required in the application itself.
You will need to:

* Provide a way to set tracing to output in json mode
* Handle panics as a `tracing::error!` in json mode.
* Ensure SIGTERM cleanly shutsdown the application and exits with a return code

The `cooldb` example crate is a complete demonstration of all these requirements.
