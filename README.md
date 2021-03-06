# Goose

Have you ever been attacked by a goose?

[![crates.io](https://img.shields.io/crates/v/goose.svg)](https://crates.io/crates/goose)
[![Documentation](https://docs.rs/goose/badge.svg)](https://docs.rs/goose)
[![Apache-2.0 licensed](https://img.shields.io/crates/l/goose.svg)](./LICENSE)
[![CI](https://github.com/tag1consulting/goose/workflows/CI/badge.svg)](https://github.com/tag1consulting/goose/actions?query=workflow%3ACI)

## Overview

Goose is a Rust load testing tool inspired by [Locust](https://locust.io/).
User behavior is defined with standard Rust code. Load tests are applications
that have a dependency on the Goose library. Web requests are made with the
[Reqwest](https://docs.rs/reqwest) HTTP Client.

## Getting Started

The 
[in-line documentation](https://docs.rs/goose/*/goose/#creating-a-simple-goose-load-test)
offers much more detail about Goose specifics. For a general background to help you get
started with Rust and Goose, read on.

[Cargo](https://doc.rust-lang.org/cargo/) is the Rust package manager. To create a new
load test, use Cargo to create a new application (you can name your application anything,
we've generically selected `loadtest`):

```bash
$ cargo new loadtest
     Created binary (application) `loadtest` package
$ cd loadtest/
```

This creates a new directory named `loadtest/` containing `loadtest/Cargo.toml` and
`loadtest/src/main.rs`. Start by editing `Cargo.toml` adding Goose under the dependencies
heading:


```toml
[dependencies]
goose = "^0.9"
```

At this point it's possible to compile all dependencies, though the
resulting binary only displays "Hello, world!":

```
$ cargo run
    Updating crates.io index
  Downloaded goose v0.9.0
      ...
   Compiling goose v0.9.0
   Compiling loadtest v0.1.0 (/home/jandrews/devel/rust/loadtest)
    Finished dev [unoptimized + debuginfo] target(s) in 52.97s
     Running `target/debug/loadtest`
Hello, world!
```

To create an actual load test, you first have to add the following boilerplate
to the top of `src/main.rs`:

```rust
use goose::prelude::*;
```

Then create a new load testing function. For our example we're simply going
to load the front page of the website we're load-testing. Goose passes all
load testing functions a mutable pointer to a GooseUser object, which is used
to track statistics and make web requests. Thanks to the Reqwest library, the
Goose client manages things like cookies, headers, and sessions for you. Load
testing functions must be declared async.

In load tests functions you typically do not set the host, and instead configure
the host at run time, so you can easily run your load test against different
environments without recompiling:

```rust
async fn loadtest_index(user: &GooseUser) -> GooseTaskResult {
    let _goose = user.get("/").await?;
}
```

Finally, edit the `main()` function, setting a return type and replacing the hello
world text as follows:

```rust
fn main() -> Result<(), GooseError> {
    GooseAttack::initialize()?
        .register_taskset(taskset!("LoadtestTasks")
            .register_task(task!(loadtest_index))
        )
        .execute()?
        .display();
    
    Ok(())
}
```

If you're new to Rust, `main()`'s return type of `Result<(), GooseError>` may look
strange. It essentially says that `main` will return nothing (`()`) on success, and
will return a `GooseError` on failure. This is helpful as several of `GooseAttack`'s
methods can fail, returning an error. In our example, `initialize()` and `execute()`
each may fail. The `?` that follows the method's name tells our program to exit and
return an error on failure, otherwise continue on. The `display()` method consumes
everything returned by `GooseAttack` and prints a summary if statistics are enabled.
The final line, `Ok(())` returns the empty result expected on success.

And that's it, you've created your first load test! Let's run it and see what
happens.

```bash
$ cargo run
   Compiling loadtest v0.1.0 (/home/jandrews/devel/rust/loadtest)
    Finished dev [unoptimized + debuginfo] target(s) in 3.56s
     Running `target/debug/loadtest`
Error: InvalidOption { option: "--host", value: "", detail: Some("host must be defined via --host, GooseAttack.set_host() or GooseTaskSet.set_host() (no host defined for WebsiteUser)") }
```

Goose is unable to run, as it doesn't know the domain you want to load test. So,
let's try again, this time passing in the `--host` flag. After running for a few
seconds, we then press `ctrl-c` to stop Goose:

```bash
$ cargo run -- --host http://apache.fosciana/
    Finished dev [unoptimized + debuginfo] target(s) in 0.07s
     Running `target/debug/loadtest --host 'http://apache.fosciana/'`
^C12:12:47 [ WARN] caught ctrl-c, stopping...
------------------------------------------------------------------------------ 
 Name                    | # reqs         | # fails        | req/s  | fail/s
 ----------------------------------------------------------------------------- 
 GET /                   | 905            | 0 (0%)         | 301    | 0    
-------------------------------------------------------------------------------
 Name                    | Avg (ms)   | Min        | Max        | Median    
 ----------------------------------------------------------------------------- 
 GET /                   | 3139       | 952        | 102412     | 3000      
-------------------------------------------------------------------------------
 Slowest page load within specified percentile of requests (in ms):
 ------------------------------------------------------------------------------
 Name                    | 50%    | 75%    | 98%    | 99%    | 99.9%  | 99.99%
 ----------------------------------------------------------------------------- 
 GET /                   | 3000   | 4000   | 5000   | 6000   | 8000   |   8000
```

When printing statistics, Goose displays three tables. The first shows the total
number of requests made (905), how many of those failed (0), the everage number
of requests per second (301), and the average number of failed requests per
second (0).

The second table shows the average time required to load a page (3139 milliseconds),
the mininimum time to load a page (952 ms), the maximum time to load a page (102412
ms) and the median time to load a page (3000 ms).

The final table shows the slowest page load time for a range of percentiles. In our
example, in the 50% fastest page loads, the slowest page loaded in 3000 ms. In the
75% fastest page loads, the slowest page loadd in 4000 ms, etc.

In most load tests you'll make have different tasks being run, and each will be
split out in the statistics, along with a line showing all totaled together in
aggregate.

Refer to the
[examples directory](https://github.com/tag1consulting/goose/tree/master/examples)
for more complicated and useful load test examples.

## Tips

* Avoid `unwrap()` in your task functions -- Goose generates a lot of load, and this tends
to trigger errors. Embrace Rust's warnings and properly handle all possible errors, this
will save you time debugging later.
* When running your load test for real, use the cargo `--release` flag to generate
optimized code. This can generate considerably more load test traffic.

## Simple Example

The `-h` flag will show all run-time configuration options available to Goose
load tests. For example, pass the `-h` flag to the `simple` example,
`cargo run --example simple -- -h`:

```
Goose 0.9.0
CLI options available when launching a Goose load test

USAGE:
    simple [FLAGS] [OPTIONS]

FLAGS:
    -h, --help             Prints help information
    -l, --list             Shows list of all possible Goose tasks and exits
    -g, --log-level        Log level (-g, -gg, -ggg, etc.)
        --manager          Enables manager mode
        --no-hash-check    Ignore worker load test checksum
        --no-stats         Don't print stats in the console
        --only-summary     Only prints summary stats
        --reset-stats      Resets statistics once hatching has been completed
        --status-codes     Includes status code counts in console stats
        --sticky-follow    User follows redirect of base_url with subsequent requests
    -V, --version          Prints version information
    -v, --verbose          Debug level (-v, -vv, -vvv, etc.)
        --worker           Enables worker mode

OPTIONS:
    -d, --debug-log-file <debug-log-file>          Debug log file name [default: ]
        --debug-log-format <debug-log-format>      Debug log format ('json' or 'raw') [default: json]
        --expect-workers <expect-workers>
            Required when in manager mode, how many workers to expect [default: 0]

    -r, --hatch-rate <hatch-rate>                  How many users to spawn per second [default: 1]
    -H, --host <host>                              Host to load test, for example: http://10.21.32.33 [default: ]
        --log-file <log-file>                      Log file name [default: goose.log]
        --manager-bind-host <manager-bind-host>    Define host manager listens on, formatted x.x.x.x [default: 0.0.0.0]
        --manager-bind-port <manager-bind-port>    Define port manager listens on [default: 5115]
        --manager-host <manager-host>              Host manager is running on [default: 127.0.0.1]
        --manager-port <manager-port>              Port manager is listening on [default: 5115]
    -t, --run-time <run-time>                      Stop after e.g. (300s, 20m, 3h, 1h30m, etc.) [default: ]
    -s, --stats-log-file <stats-log-file>          Statistics log file name [default: ]
        --stats-log-format <stats-log-format>      Statistics log format ('csv', 'json', or 'raw') [default: json]
        --throttle-requests <throttle-requests>    Throttle (max) requests per second
    -u, --users <users>                            Number of concurrent Goose users (defaults to available CPUs)
```

The `examples/simple.rs` example copies the simple load test documented on the locust.io web page, rewritten in Rust for Goose. It uses minimal advanced functionality, but demonstrates how to GET and POST pages. It defines a single Task Set which has the user log in and then load a couple of pages.

Goose can make use of all available CPU cores. By default, it will launch 1 user per core, and it can be configured to launch many more. The following was configured instead to launch 1,024 users. Each user randomly pauses 5 to 15 seconds after each task is loaded, so it's possible to spin up a large number of users. Here is a snapshot of `top` when running this example on an 8-core VM with 10G of available RAM -- there were ample resources to launch considerably more "users", though `ulimit` had to be resized:

```
top - 11:14:57 up 16 days,  4:40,  2 users,  load average: 0.00, 0.04, 0.01
Tasks: 129 total,   1 running, 128 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.3 sy,  0.0 ni, 99.2 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :   9993.6 total,   6695.1 free,   1269.3 used,   2029.2 buff/cache
MiB Swap:  10237.0 total,  10234.7 free,      2.3 used.   8401.5 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                   
19776 goose     20   0    9.8g 874688   8252 S   6.3   8.5   0:42.90 simple                                                    
```

Here's the output of running the loadtest. The `-v` flag sends `INFO` and more critical messages to stdout (in addition to the log file). The `-u1024` tells Goose to spin up 1,024 users. The `-r32` option tells Goose to spin up 32 users per second. The `-t 10m` option tells Goose to run the load test for 10 minutes, or 600 seconds. The `--print-stats` flag tells Goose to collect statistics during the load test, and the `--status-codes` flag tells it to include statistics about HTTP Status codes returned by the server. Finally, the `--only-summary` flag tells Goose to only display the statistics when the load test finishes, otherwise it would display running statistics every 15 seconds for the duration of the test.

```
$ cargo run --release --example simple -- --host http://apache.fosciana -v -u1024 -r32 -t 10m --print-stats --status-codes --only-summary
    Finished release [optimized] target(s) in 0.05s
     Running `target/release/examples/simple --host 'http://apache.fosciana' -v -u1024 -r32 -t 10m --print-stats --status-codes --only-summary`
18:42:48 [ INFO] Output verbosity level: INFO
18:42:48 [ INFO] Logfile verbosity level: INFO
18:42:48 [ INFO] Writing to log file: goose.log
18:42:48 [ INFO] run_time = 600
18:42:48 [ INFO] global host configured: http://apache.fosciana
18:42:53 [ INFO] launching user 1 from WebsiteUser...
18:42:53 [ INFO] launching user 2 from WebsiteUser...
18:42:53 [ INFO] launching user 3 from WebsiteUser...
18:42:53 [ INFO] launching user 4 from WebsiteUser...
18:42:53 [ INFO] launching user 5 from WebsiteUser...
18:42:53 [ INFO] launching user 6 from WebsiteUser...
18:42:53 [ INFO] launching user 7 from WebsiteUser...
18:42:53 [ INFO] launching user 8 from WebsiteUser...

```
...
```
18:43:25 [ INFO] launching user 1022 from WebsiteUser...
18:43:25 [ INFO] launching user 1023 from WebsiteUser...
18:43:25 [ INFO] launching user 1024 from WebsiteUser...
18:43:25 [ INFO] launched 1024 users...
18:53:26 [ INFO] stopping after 600 seconds...
18:53:26 [ INFO] waiting for users to exit
------------------------------------------------------------------------------ 
 Name                    | # reqs         | # fails        | req/s  | fail/s
 ----------------------------------------------------------------------------- 
 GET /                   | 34,077         | 582 (1.7%)     | 53     | 0    
 GET /about/             | 34,044         | 610 (1.8%)     | 53     | 0    
 POST /login             | 1,024          | 0 (0%)         | 1      | 0    
 ------------------------+----------------+----------------+-------+---------- 
 Aggregated              | 69,145         | 1,192 (1.7%)   | 107    | 1    
-------------------------------------------------------------------------------
 Name                    | Avg (ms)   | Min        | Max        | Median      
 ----------------------------------------------------------------------------- 
 GET /                   | 12.38      | 0.01       | 1001.10    | 0.09      
 GET /about/             | 12.80      | 0.01       | 1001.10    | 0.08      
 POST /login             | 0.21       | 0.15       | 1.82       | 0.20      
 ------------------------+------------+------------+------------+------------- 
 Aggregated              | 12.41      | 0.01       | 1001.10    | 0.02      
-------------------------------------------------------------------------------
 Slowest page load within specified percentile of requests (in ms):
 ------------------------------------------------------------------------------
 Name                    | 50%    | 75%    | 98%    | 99%    | 99.9%  | 99.99%
 ----------------------------------------------------------------------------- 
 GET /                   | 0.09   | 0.10   | 345.18 | 500.60 | 1000.93 | 1001.09
 GET /about/             | 0.08   | 0.10   | 356.65 | 500.61 | 1000.94 | 1001.08
 POST /login             | 0.20   | 0.22   | 0.27   | 0.34   | 1.36   |   1.82
 ------------------------+------------+------------+------------+------------- 
 Aggregated              | 0.08   | 0.10   | 349.40 | 500.60 | 1000.93 | 1001.09
-------------------------------------------------------------------------------
 Name                    | Status codes              
 ----------------------------------------------------------------------------- 
 GET /                   | 33,495 [200], 582 [0]      
 GET /about/             | 33,434 [200], 610 [0]      
 POST /login             | 1,024 [200]              
-------------------------------------------------------------------------------
 Aggregated              | 67,953 [200]              
```

## Throttling Requests

By default, Goose will generate as much load as it can. If this is not desirable, the
throttle allows optionally limiting the maximum number of requests per second made during
a load test. This can be helpful to ensure consistency when running a load test from
multiple different servers with different available resources.

The throttle is specified as an integer. For example:

```rust
$ cargo run --example simple -- --host http://local.dev/ -u100 -r20 -v --throttle-requests 5
```

In this example, Goose will launch 100 GooseUser threads, but the throttle will prevent them from
generating a combined total of more than 5 requests per second. The `--throttle-requests` command
line option imposes a maximum number of requests, not a minimum number of requests.

## Logging Load Test Requests

Goose can optionally log details about all load test requests to a file. To enable, add
the `--stats-log-file=foo` command line option, where `foo` is either a relative or
absolute path of the log file to create. Any existing file that may already exist will be
overwritten.

When operating in Gaggle-mode, the `--stats-log-file` option can be enabled on worker
processes and/or on the manager process. You can therefor configure Goose to spread out
the overhead of writing logs by enabling the option on workers, or you can configure
Goose to create a single combined log of all requests by enabling the option on the
manager.

By default, logs are written in JSON Lines format. For example:

```json
{"elapsed":30,"final_url":"http://local.dev/user/42","method":"POST","name":"/login","redirected":true,"response_time":220,"status_code":200,"success":true,"update":false,"url":"http://local.dev/login","user":0}
{"elapsed":251,"final_url":"http://local.dev/","method":"GET","name":"/","redirected":false,"response_time":3,"status_code":200,"success":true,"update":false,"url":"http://local.dev/","user":0}
{"elapsed":1027,"final_url":"http://local.dev/user/13","method":"POST","name":"/login","redirected":true,"response_time":266,"status_code":200,"success":true,"update":false,"url":"http://local.dev/login","user":1}
{"elapsed":1294,"final_url":"http://local.dev/","method":"GET","name":"/","redirected":false,"response_time":4,"status_code":200,"success":true,"update":false,"url":"http://local.dev/","user":1}
```

Logs include the entire `GooseRawRequest` object as defined in `src/goose.rs`, which
are created on all requests. This object includes the following fields:
 - `elapsed`: total milliseconds between when this `GooseUser` thread started and this
   request was made;
 - `method`: the type of HTTP request made;
 - `name`: the name of the request;
 - `url`: the URL that was requested;
 - `final_url`: the URL that was returned;
 - `redirected`: true or false if the request was redirected;
 - `response_time`: how many milliseconds the request took;
 - `status_code`: the HTTP response code returned for this request;
 - `success`: true or false if this was a successful request;
 - `update`: true or false if this is a recurrence of a previous log entery, but with
   `success` toggling between `true` and `false`. This happens when a load test calls
   `set_success()` on a request that Goose previously interpreted as a failure, or
   `set_failure()` on a request that Goose interpreted as a success;
 - `user`: an integer value indicating which `GooseUser` thread made this request.

In the first line of the above example, `GooseUser` thread 0 made a `POST` request to
`/login` and was successfully redirected to `/user/42` in 220 milliseconds. The second
line is the same `GooseUser` thread which then made a `GET` request to `/` in 3
milliseconds. The third and fourth lines are a second `GooseUser` thread doing the same
thing, first logging in and then loading the front page.

By default Goose logs statistics in JSON Lines format. The `--stats-log-format` option
can be used to log in `csv`, `json` or `raw` format. The `raw` format is Rust's debug
output of the entire `GooseRawRequest` object.

For example, `csv` output of the same requests logged above would look like:
```csv
elapsed,method,name,url,final_url,redirected,response_time,status_code,success,update,user
30,POST,"/login","http://local.dev/login","http://local.dev/user/42",true,30,200,true,false,0
251,GET,"/","http://local.dev/","http://local.dev/",false,3,200,true,false,0
1027,POST,"/login","http://local.dev/login","http://local.dev/user/13",true,266,200,true,false,1
1294,GET,"/","http://local.dev/","http://local.dev/",false,4,200,true,false,1
```

## Load Test Debug Logging

Goose can optionally log details about requests and responses for debug purposes. When writing
a load test you must invoke `client.log_debug(tag, Option<request>, Option<headers>, Option<body>)`
where `tag` is an arbitrary string to identify where in the load test and/or why debug is being
written, `request` is a `GooseRawRequest` object, `headers` are the HTTP headers returned by the
server, and `body` is the web page body returned by the server.

For an example on how to correctly use `client.log_debug()`, including how to obtain the response
headers and body, see `examples/drupal_loadtest`.

If the load test is run with the `--debug-log-file=foo` command line option, where `foo` is either
a relative or an absolute path, Goose will log all debug generated by calls to `client.log_debug()`
to this file. Debug is logged in JSON Lines format. For example:

```json
{"body":"<!DOCTYPE html>\n<html>\n  <head>\n    <title>503 Backend fetch failed</title>\n  </head>\n  <body>\n    <h1>Error 503 Backend fetch failed</h1>\n    <p>Backend fetch failed</p>\n    <h3>Guru Meditation:</h3>\n    <p>XID: 923425</p>\n    <hr>\n    <p>Varnish cache server</p>\n  </body>\n</html>\n","header":"{\"date\": \"Wed, 01 Jul 2020 10:27:31 GMT\", \"server\": \"Varnish\", \"content-type\": \"text/html; charset=utf-8\", \"retry-after\": \"5\", \"x-varnish\": \"923424\", \"age\": \"0\", \"via\": \"1.1 varnish (Varnish/6.1)\", \"x-varnish-cache\": \"MISS\", \"x-varnish-cookie\": \"SESSd7e04cba6a8ba148c966860632ef3636=hejsW1mQnnsHlua0AicCjEpUjnCRTkOLubwL33UJXRU\", \"content-length\": \"283\", \"connection\": \"keep-alive\"}","request":{"elapsed":4192,"final_url":"http://local.dev/node/3247","method":"GET","name":"(Auth) comment form","redirected":false,"response_time":8,"status_code":503,"success":false,"update":false,"url":"http://local.dev/node/3247","user":4},"tag":"post_comment: no form_build_id found on node/3247"}
```

If `--debug-log-file=foo` is not specified at run time, nothing will be logged.

By default Goose writes debug logs in JSON Lines format. The `--debug-log-format` option
can be used to log in `json` or `raw` format. The `raw` format is Rust's debug
output of the entire `GooseDebug` object.

## Gaggle: Distributed Load Test

Goose also supports distributed load testing. A Gaggle is one Goose process
running in manager mode, and 1 or more Goose processes running in worker mode.
The manager coordinates starting and stopping the workers, and collects
aggregated statistics. Gaggle support is a cargo feature that must be enabled
at compile-time as documented below. To launch a Gaggle, you must copy your
load test application to all servers from which you wish to generate load.

### Gaggle Compile-time Feature

Gaggle support is a compile-time Cargo feature that must be enabled. Goose uses
the [`nng`](https://docs.rs/nng/) library to manage network connections, and
compiling `nng` requires that `cmake` be available.

The `gaggle` feature can be enabled from the command line by adding
`--features gaggle` to your cargo command.

When writing load test applications, you can default to compiling in the Gaggle
feature in the `dependencies` section of your `Cargo.toml`, for example:

```toml
[dependencies]
goose = { version = "^0.9", features = ["gaggle"] }
```

### Goose Manager

To launch a Gaggle, you first must start a Goose application in manager
mode. All configuration happens in the manager. To start, add the `--manager`
flag and the `--expect-workers` flag, the latter necessary to tell the Manager
process how many Worker processes it will be coordinating. For example:

```
cargo run --features gaggle --example simple -- --manager --expect-workers 2 --host http://local.dev/ -v
```

This configures a Goose manager to listen on all interfaces on the default
port (0.0.0.0:5115) for 2 Goose worker processes.

### Goose Worker

At this time, a Goose process can be either a manager or a worker, not both.
Therefor, it makes sense to launch your first worker on the same server that
the manager is running on. If not otherwise configured, a Goose worker will
try to connect to the manager on the localhost. This can be done as folows:

```
cargo run --features gaggle --example simple -- --worker -v
```

In our above example, we expected 2 workers. The second Goose process should
be started on a different server. This will require telling it the host where
the Goose manager proocess is running. For example:

```
cargo run --example simple -- --worker --manager-host 192.168.1.55 -v
```

Once all expected workers are running, the distributed load test will
automatically start. We set the `-v` flag so Goose provides verbose output
indicating what is happening. In our example, the load test will run until
it is canceled. You can cancel the manager or either of the worker processes,
and the test will stop on all servers.

### Goose Run-time Flags

* `--manager`: starts a Goose process in manager mode. There currently can only be one manager per Gaggle.
* `--worker`: starts a Goose process in worker mode. How many workers are in a given Gaggle is defined by the `--expect-workers` option, documented below.
* `--no-hash-check`: tells Goose to ignore if the load test applications don't match between worker(s) and manager. Not recommended.

The `--no-stats`, `--only-summary`, `--reset-stats`, `--status-codes`, and `--no-hash-check` flags must be set on the manager. Workers inheret these flags from the manager

### Goose Run-time Options

* `--manager-bind-host <manager-bind-host>`: configures the host that the manager listens on. By default Goose will listen on all interfaces, or `0.0.0.0`.
* `--manager-bind-port <manager-bind-port>`: configures the port that the manager listens on. By default Goose will listen on port `5115`.
* `--manager-host <manager-host>`: configures the host that the worker will talk to the manager on. By default, a Goose worker will connect to the localhost, or `127.0.0.1`. In a distributed load test, this must be set to the IP of the Goose manager.
* `--manager-port <manager-port>`: configures the port that a worker will talk to the manager on. By default, a Goose worker will connect to port `5115`.

The `--users`, `--hatch-rate`, `--host`, and `--run-time` options must be set on the manager. Workers inheret these options from the manager.

The `--throttle-requests` option must be configured on each worker, and can be set to a different value
on each worker if desired.

### Technical Details

Goose uses [`nng`](https://docs.rs/nng/) to send network messages between
the manager and all workers. [Serde](https://docs.serde.rs/serde/index.html)
and [Serde CBOR](https://github.com/pyfisch/cbor) are used to serialize messages
into [Concise Binary Object Representation](https://tools.ietf.org/html/rfc7049).

Workers initiate all network connections, and push a HashMap containing load test
statistics up to the manager process.

## RustLS

By default Reqwest (and therefore Goose) uses the system-native transport layer security to
make HTTPS requests. This means schannel on Windows, Security-Framework on macOS, and OpenSSL
on Linux. If you'd prefer to use a [pure Rust TLS implementation](https://github.com/ctz/rustls),
disable default features and enable `rustls` in `Cargo.toml` as follows:

```toml
[dependencies]
goose = { version = "^0.9", default-features = false, features = ["rustls"] }
```

## Roadmap

The Goose project roadmap is documented in [TODO.md](https://github.com/tag1consulting/goose/blob/master/TODO.md).
