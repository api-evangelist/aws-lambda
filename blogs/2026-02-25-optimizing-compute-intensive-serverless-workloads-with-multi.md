---
title: "Optimizing Compute-Intensive Serverless Workloads with Multi-threaded Rust on AWS Lambda"
url: "https://aws.amazon.com/blogs/compute/optimizing-compute-intensive-serverless-workloads-with-multi-threaded-rust-on-aws-lambda/"
date: "Wed, 25 Feb 2026 12:49:44 +0000"
author: "Daniel Abib"
feed_url: "https://aws.amazon.com/blogs/compute/category/compute/aws-lambda/feed/"
---
<p>Customers use <a href="https://aws.amazon.com/lambda/">AWS Lambda</a> to build Serverless applications for a wide variety of use cases, from simple API backends to complex data processing pipelines. Lambda’s flexibility makes it an excellent choice for many workloads, and with support for up to 10,240 MB of memory, you can now tackle compute-intensive tasks that were previously challenging in a Serverless environment. When you configure a Lambda function’s memory size, you allocate RAM and Lambda automatically provides proportional CPU power. When you configure 10,240 MB, your Lambda function has access to up to 6 vCPUs.</p> 
<p>However, there’s an important consideration that many developers discover: <strong>simply allocating more memory may not automatically make your function faster.</strong> If your code runs sequentially, it will only use one vCPU regardless of how many are available. The remaining vCPUs sit idle while you’re still paying for the full memory allocation.</p> 
<p>To help benefit from Lambda’s multi-core capabilities, your code should explicitly implement concurrent processing through multi-threading or parallel execution. Without this, you’re paying for compute power you’re not using.</p> 
<p>Rust provides excellent support for this pattern. The <a href="https://github.com/aws/aws-lambda-rust-runtime">AWS Lambda Rust Runtime</a> provides developers with a language that combines exceptional performance with built-in concurrency primitives. In this post, we show you how to implement multi-threading in Rust to achieve 4-6x performance improvements for CPU-intensive workloads.</p> 
<h2>Our Test Workload: Why Bcrypt Password Hashing?</h2> 
<p>For this analysis, we use <strong>bcrypt password hashing</strong> as our CPU-intensive workload to evaluate multi-core scaling behavior. This choice is deliberate for several reasons:</p> 
<ol> 
 <li><strong>Real-world relevance</strong>: Bcrypt is commonly used in authentication systems, making our benchmarks practically relevant rather than synthetic.</li> 
 <li><strong>Predictable CPU work</strong>: Bcrypt with cost factor 10 provides approximately 100ms of pure CPU work per operation on typical hardware, creating a consistent and measurable baseline.</li> 
 <li><strong>Embarrassingly parallel</strong>: Each hash operation is completely independent, making it an ideal candidate for parallel processing without shared state or lock contention.</li> 
 <li><strong>CPU-bound</strong>: Bcrypt is deterministic and CPU-bound (not memory or I/O bound), isolating the performance characteristics we want to measure.</li> 
</ol> 
<p>Throughout this post, we process batches of passwords and measure how multi-threading improves throughput as we scale from 1 to 6 vCPUs.</p> 
<h2>Understanding Lambda’s vCPU Allocation</h2> 
<p>AWS Lambda allocates CPU resources proportionally to the configured memory. According to <a href="https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html">AWS Lambda function memory documentation</a>, at 1,769 MB a function has the equivalent of one vCPU.</p> 
<p style="text-align: center;"><a href="https://www.youtube.com/watch?v=aW5EtKHTMuQ&amp;t=339s"><strong>vCPU Allocation by Memory</strong></a><strong>:</strong></p> 
<table style="margin: 0px auto; height: 258px;" width="335"> 
 <thead> 
  <tr> 
   <td> <p style="text-align: center;">Memory (MB)</p> </td> 
   <td style="text-align: center;">Approximate vCPUs</td> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td style="text-align: center;">128 – 1,769</td> 
   <td style="text-align: center;">~1</td> 
  </tr> 
  <tr> 
   <td style="text-align: center;">1,770 – 3,538</td> 
   <td style="text-align: center;">~2</td> 
  </tr> 
  <tr> 
   <td style="text-align: center;">3,539 – 5,307</td> 
   <td style="text-align: center;">~3</td> 
  </tr> 
  <tr> 
   <td style="text-align: center;">5,308 – 7,076</td> 
   <td style="text-align: center;">~4</td> 
  </tr> 
  <tr> 
   <td style="text-align: center;">7,077 – 8,845</td> 
   <td style="text-align: center;">~5</td> 
  </tr> 
  <tr> 
   <td style="text-align: center;">8,846 – 10,240</td> 
   <td> <p style="text-align: center;">~6</p> </td> 
  </tr> 
 </tbody> 
</table> 
<p><strong>Note</strong>: The <code>num_cpus</code> crate returns the number of logical CPUs visible to the Lambda environment, which may differ from the allocated vCPU share. At lower memory configurations, you may see 2 CPUs reported even though only 1 vCPU worth of compute time is allocated.</p> 
<h2>Solution Overview</h2> 
<p>The solution consists of a Rust Lambda function that:</p> 
<ol> 
 <li>Receives a request specifying the number of items to process</li> 
 <li>Detects available vCPUs and configures a thread pool accordingly</li> 
 <li>Processes items in parallel using the <a href="https://github.com/rayon-rs/rayon">Rayon library</a> (a data parallelism library that allows you to convert sequential iterators into parallel ones with a <code>.par_iter()</code> call)</li> 
 <li>Returns performance metrics including duration and throughput</li> 
</ol> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/02/24/Picture1-6.png"><img alt="" class="aligncenter wp-image-25731 size-large" height="1024" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/02/24/Picture1-6-683x1024.png" width="683" /></a></p> 
<p style="text-align: center;"><em>Architecture Diagram: Lambda receives request, initializes Rayon thread pool based on <code>WORKER_COUNT</code> environment variable, processes bcrypt hashes in parallel across multiple vCPUs, and returns results.</em></p> 
<h2>Creating a Multi-threaded Rust Lambda Function</h2> 
<p>Create a new Lambda project using Cargo Lambda:</p> 
<pre><code class="language-bash">cargo lambda new rust-multithread-demo
cd rust-multithread-demo</code></pre> 
<h3>Dependencies</h3> 
<p>Update <code>Cargo.toml</code> with the necessary dependencies:</p> 
<pre><code class="language-toml">[package]
name = "rust-multithread-lambda"
version = "0.1.0"
edition = "2021"

[dependencies]
lambda_runtime = "1.0.0"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
bcrypt = "0.15"
rayon = "1.7"
num_cpus = "1.16"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
strip = true</code></pre> 
<p>The optimization flags in <code>[profile.release]</code> reduce binary size and improve performance:</p> 
<ul> 
 <li><code>opt-level = 3</code>: Maximum optimization</li> 
 <li><code>lto = true</code>: Link-time optimization for smaller binaries</li> 
 <li><code>strip = true</code>: Remove debug symbols</li> 
</ul> 
<h3>Implementing the Lambda Entry Point</h3> 
<p>First, let’s look at how we initialize the thread pool during cold start:</p> 
<p><strong>src/main.rs</strong>:</p> 
<pre><code class="language-rust">use lambda_runtime::{run, service_fn, Error, LambdaEvent};
mod handler;
use handler::{function_handler, get_worker_count, init_thread_pool, ProcessRequest};

#[tokio::main]
async fn main() -&gt; Result&lt;(), Error&gt; {
    // Initialize Rayon thread pool at cold start (once per container lifecycle)
    init_thread_pool(get_worker_count());

    run(service_fn(|event: LambdaEvent&lt;ProcessRequest&gt;| async move {
        function_handler(event.payload).await
    }))
    .await
}</code></pre> 
<p><strong>Why initialize in <code>main()</code> and not in the handler?</strong></p> 
<ol> 
 <li><strong>Deterministic Configuration</strong>: The thread pool is configured once per container, before any requests arrive. This prevents race conditions if multiple requests try to initialize concurrently.</li> 
 <li><strong>Container Reuse</strong>: Lambda containers can serve multiple requests. Initializing in <code>main()</code> ensures the configuration is set during the cold start and persists for all subsequent warm invocations.</li> 
 <li><strong>Performance</strong>: Thread pool setup happens during cold start (already counted as initialization time), not during request processing.</li> 
</ol> 
<h3>Implementing the Request Handler</h3> 
<p><strong>src/handler.rs</strong>:</p> 
<pre><code class="language-rust">use serde::{Deserialize, Serialize};
use std::env;
use std::sync::Once;
use std::time::Instant;
use std::collections::HashSet;
use std::sync::Mutex;
use rayon::prelude::*;

static INIT: Once = Once::new();

#[derive(Deserialize)]
pub struct ProcessRequest {
    count: usize,
    mode: String,
}

#[derive(Serialize)]
pub struct ProcessResponse {
    processed: usize,
    duration_ms: u128,
    mode: String,
    workers: usize,
    detected_cpus: usize,
    avg_ms_per_item: f64,
    memory_used_kb: u64,
    threads_used: usize, // Actual threads that processed items (proves multi-threading)
}

// CPU-intensive bcrypt hashing with cost factor 10
fn hash_password(password: &amp;str) -&gt; Result&lt;String, bcrypt::BcryptError&gt; {
    bcrypt::hash(password, 10)
}

// Process items one at a time (baseline for comparison)
fn process_sequential(items: Vec&lt;String&gt;) -&gt; Result&lt;(Vec&lt;String&gt;, usize), Box&lt;dyn std::error::Error + Send + Sync&gt;&gt; {
    let results: Result&lt;Vec&lt;String&gt;, _&gt; = items
        .iter()
        .map(|item| hash_password(item))
        .collect();
    results
        .map(|r| (r, 1))
        .map_err(|e| Box::new(e) as Box&lt;dyn std::error::Error + Send + Sync&gt;)
}

// Process items in parallel using Rayon's work-stealing scheduler
// Thread pool size is configured once at cold start via init_thread_pool()
fn process_parallel(items: Vec&lt;String&gt;) -&gt; Result&lt;(Vec&lt;String&gt;, usize), Box&lt;dyn std::error::Error + Send + Sync&gt;&gt; {
    let thread_ids: Mutex&lt;HashSet&lt;std::thread::ThreadId&gt;&gt; = Mutex::new(HashSet::new());

    let results: Result&lt;Vec&lt;String&gt;, _&gt; = items
        .par_iter()
        .map(|item| {
            thread_ids.lock().unwrap().insert(std::thread::current().id());
            hash_password(item)
        })
        .collect();

    let threads_used = thread_ids.lock().unwrap().len();
    results
        .map(|r| (r, threads_used))
        .map_err(|e| Box::new(e) as Box&lt;dyn std::error::Error + Send + Sync&gt;)
}

// Get worker count from env var or detect CPUs, clamped to 1-6
pub fn get_worker_count() -&gt; usize {
    if let Ok(count_str) = env::var("WORKER_COUNT") {
        if let Ok(count) = count_str.parse::&lt;usize&gt;() {
            return count.clamp(1, 6);
        }
    }
    num_cpus::get().clamp(1, 6)
}

// Initialize Rayon global thread pool (only once per Lambda container)
pub fn init_thread_pool(workers: usize) {
    INIT.call_once(|| {
        let _ = rayon::ThreadPoolBuilder::new()
            .num_threads(workers)
            .build_global();
    });
}

// Read RSS memory from /proc/self/statm (Linux only)
fn get_memory_usage_kb() -&gt; u64 {
    std::fs::read_to_string("/proc/self/statm")
        .ok()
        .and_then(|s| s.split_whitespace().nth(1)?.parse::&lt;u64&gt;().ok())
        .map(|pages| pages * 4)
        .unwrap_or(0)
}

// Main Lambda handler - processes items sequentially or in parallel
pub async fn function_handler(request: ProcessRequest) -&gt; Result&lt;ProcessResponse, Box&lt;dyn std::error::Error + Send + Sync&gt;&gt; {
    if request.count == 0 { return Err("count must be greater than 0".into()); }
    if request.count &gt; 1000 { return Err("count exceeds maximum of 1000 items".into()); }

    let items: Vec&lt;String&gt; = (0..request.count)
        .map(|i| format!("password_{:06}", i))
        .collect();

    let workers = get_worker_count();
    let mode = match request.mode.as_str() {
        "sequential" =&gt; "sequential",
        "parallel"   =&gt; "parallel",
        _            =&gt; if workers &gt; 1 { "parallel" } else { "sequential" },
    };

    let start = Instant::now();
    let (results, threads_used) = match mode {
        "sequential" =&gt; process_sequential(items)?,
        _            =&gt; process_parallel(items)?,
    };
    let duration_ms = start.elapsed().as_millis();

    Ok(ProcessResponse {
        processed: results.len(),
        duration_ms,
        mode: mode.to_string(),
        workers: if mode == "parallel" { workers } else { 1 },
        detected_cpus: num_cpus::get(),
        avg_ms_per_item: duration_ms as f64 / request.count as f64,
        memory_used_kb: get_memory_usage_kb(),
        threads_used,
    })
}</code></pre> 
<h3>Key Implementation Details</h3> 
<p><strong>Thread Pool Initialization at Cold Start</strong>: The code initializes the thread pool in <code>main()</code> before the Lambda runtime starts, not during request processing. This approach is designed to eliminate race conditions and provide deterministic behavior across all invocations.</p> 
<p><strong>Important Note</strong>: Lambda initializes the thread pool once per container. The thread pool configuration retains its original value even if you change the <code>WORKER_COUNT</code> environment variable between invocations within the same container. For production deployments, keep <code>WORKER_COUNT</code> consistent for the function’s lifecycle.</p> 
<p><strong>Input Validation</strong>: The handler validates that <code>count</code> is between 1 and 1000 to prevent resource exhaustion.</p> 
<p><strong>Thread Tracking</strong>: The <code>threads_used</code> field proves multi-threading is working by tracking unique thread IDs during parallel processing. This provides empirical validation that work is distributed across multiple threads.</p> 
<p><strong>Memory Tracking</strong>: The <code>memory_used_kb</code> field reports RSS memory usage by reading <code>/proc/self/statm</code>, providing visibility into actual memory consumption.</p> 
<p><strong>Mode Selection</strong>: The function supports three modes:</p> 
<ul> 
 <li><code>sequential</code>: Single-threaded processing</li> 
 <li><code>parallel</code>: Multi-threaded processing using Rayon</li> 
 <li><code>auto</code>: Automatically selects based on available workers</li> 
</ul> 
<h2>Building and Deploying</h2> 
<p>With the implementation complete, let’s compile the function for Lambda’s environment and deploy it to AWS.</p> 
<pre><code class="language-bash"># Build for ARM64 (Graviton2) - recommended for cost efficiency
cargo lambda build --release --arm64

# Or build for x86_64
cargo lambda build --release --x86-64</code></pre> 
<p>The build process produces a binary of approximately <strong>1.7 MB</strong> (uncompressed) or <strong>0.8 MB</strong> (zipped).</p> 
<h3>Deploy to AWS</h3> 
<p>Use Cargo Lambda to deploy the function with your desired memory configuration and worker count.</p> 
<pre><code class="language-bash"># Deploy with 6144 MB memory (4 vCPUs) and 4 workers
cargo lambda deploy rust-multithread-lambda \
    --memory 6144 \
    --timeout 30 \
    --env-var WORKER_COUNT=4</code></pre> 
<p><strong>Note</strong>: To test different configurations, repeat the build and deploy commands with different <code>--memory</code> values and <code>WORKER_COUNT</code> settings for each configuration you want to benchmark. For comprehensive testing across architectures, build with <code>--arm64</code>, deploy all memory configurations, then rebuild with <code>--x86-64</code> and deploy again.</p> 
<h3>Required IAM Permissions</h3> 
<p>The Lambda execution role needs the following permissions:</p> 
<pre><code class="language-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        }
    ]
}</code></pre> 
<h3>Test the Function</h3> 
<p>After deployment, verify the function works correctly by invoking it with a test payload.</p> 
<pre><code class="language-bash">aws lambda invoke \
    --function-name rust-multithread-lambda \
    --payload '{"count":20,"mode":"parallel"}' \
    --cli-binary-format raw-in-base64-out \
    response.json</code></pre> 
<h2>Performance Benchmarks</h2> 
<p>We tested multiple configurations on ARM64 (Graviton2) to measure the impact of multi-threading.</p> 
<p><strong>Test workload</strong>: Processing 20 bcrypt password hashes (cost factor 10)</p> 
<p><strong>Note</strong>: Benchmark results may vary between runs due to factors such as Lambda placement, underlying hardware differences, and AWS infrastructure conditions. The numbers presented here are representative of typical performance observed across multiple test runs.</p> 
<h3>Performance Results: ARM64 (Graviton2)</h3> 
<table width="100%"> 
 <thead> 
  <tr> 
   <td>Memory</td> 
   <td>vCPUs</td> 
   <td>Workers</td> 
   <td>Avg (ms)</td> 
   <td>P50 (ms)</td> 
   <td>P95 (ms)</td> 
   <td>P99 (ms)</td> 
   <td>Min</td> 
   <td>Max</td> 
   <td>Speedup</td> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>1536 MB</td> 
   <td>~1</td> 
   <td>1</td> 
   <td>1,885</td> 
   <td>1,882</td> 
   <td>1,898</td> 
   <td>1,898</td> 
   <td>1,877</td> 
   <td>1,907</td> 
   <td>1.00x</td> 
  </tr> 
  <tr> 
   <td>2048 MB</td> 
   <td>~2</td> 
   <td>2</td> 
   <td>1,334</td> 
   <td>1,331</td> 
   <td>1,341</td> 
   <td>1,341</td> 
   <td>1,324</td> 
   <td>1,356</td> 
   <td>1.41x</td> 
  </tr> 
  <tr> 
   <td>4096 MB</td> 
   <td>~3</td> 
   <td>3</td> 
   <td>685</td> 
   <td>683</td> 
   <td>699</td> 
   <td>699</td> 
   <td>669</td> 
   <td>704</td> 
   <td>2.75x</td> 
  </tr> 
  <tr> 
   <td>6144 MB</td> 
   <td>~4</td> 
   <td>4</td> 
   <td>463</td> 
   <td>464</td> 
   <td>467</td> 
   <td>467</td> 
   <td>453</td> 
   <td>469</td> 
   <td>4.07x</td> 
  </tr> 
  <tr> 
   <td>8192 MB</td> 
   <td>~5</td> 
   <td>5</td> 
   <td>338</td> 
   <td>343</td> 
   <td>345</td> 
   <td>345</td> 
   <td>325</td> 
   <td>346</td> 
   <td>5.57x</td> 
  </tr> 
  <tr> 
   <td>10240 MB</td> 
   <td>~6</td> 
   <td>6</td> 
   <td><strong>280</strong></td> 
   <td><strong>278</strong></td> 
   <td><strong>292</strong></td> 
   <td><strong>292</strong></td> 
   <td>271</td> 
   <td>293</td> 
   <td><strong>6.73x</strong></td> 
  </tr> 
 </tbody> 
</table> 
<h3>Performance Results: x86_64</h3> 
<table width="100%"> 
 <thead> 
  <tr> 
   <td>Memory</td> 
   <td>vCPUs</td> 
   <td>Workers</td> 
   <td>Avg (ms)</td> 
   <td>P50 (ms)</td> 
   <td>P95 (ms)</td> 
   <td>P99 (ms)</td> 
   <td>Min</td> 
   <td>Max</td> 
   <td>Speedup</td> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>1536 MB</td> 
   <td>~1</td> 
   <td>1</td> 
   <td>1,671</td> 
   <td>1,675</td> 
   <td>1,681</td> 
   <td>1,681</td> 
   <td>1,659</td> 
   <td>1,684</td> 
   <td>1.00x</td> 
  </tr> 
  <tr> 
   <td>2048 MB</td> 
   <td>~2</td> 
   <td>2</td> 
   <td>1,253</td> 
   <td>1,249</td> 
   <td>1,265</td> 
   <td>1,265</td> 
   <td>1,241</td> 
   <td>1,294</td> 
   <td>1.33x</td> 
  </tr> 
  <tr> 
   <td>4096 MB</td> 
   <td>~3</td> 
   <td>3</td> 
   <td>892</td> 
   <td>891</td> 
   <td>899</td> 
   <td>899</td> 
   <td>888</td> 
   <td>900</td> 
   <td>1.87x</td> 
  </tr> 
  <tr> 
   <td>6144 MB</td> 
   <td>~4</td> 
   <td>4</td> 
   <td>429</td> 
   <td>425</td> 
   <td>443</td> 
   <td>443</td> 
   <td>417</td> 
   <td>449</td> 
   <td>3.89x</td> 
  </tr> 
  <tr> 
   <td>8192 MB</td> 
   <td>~5</td> 
   <td>5</td> 
   <td>330</td> 
   <td>323</td> 
   <td>349</td> 
   <td>349</td> 
   <td>317</td> 
   <td>358</td> 
   <td>5.06x</td> 
  </tr> 
  <tr> 
   <td>10240 MB</td> 
   <td>~6</td> 
   <td>6</td> 
   <td>292</td> 
   <td>292</td> 
   <td>298</td> 
   <td>298</td> 
   <td>291</td> 
   <td>298</td> 
   <td>5.72x</td> 
  </tr> 
 </tbody> 
</table> 
<h3>Architecture Comparison</h3> 
<table width="100%"> 
 <thead> 
  <tr> 
   <td>Memory</td> 
   <td>Workers</td> 
   <td>ARM64 Avg</td> 
   <td>x86_64 Avg</td> 
   <td>Diff %</td> 
   <td>Faster Arch</td> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>1536 MB</td> 
   <td>1</td> 
   <td>1,885 ms</td> 
   <td>1,671 ms</td> 
   <td>-12.8%</td> 
   <td>x86_64</td> 
  </tr> 
  <tr> 
   <td>2048 MB</td> 
   <td>2</td> 
   <td>1,334 ms</td> 
   <td>1,253 ms</td> 
   <td>-6.4%</td> 
   <td>x86_64</td> 
  </tr> 
  <tr> 
   <td>4096 MB</td> 
   <td>3</td> 
   <td>685 ms</td> 
   <td>892 ms</td> 
   <td>+23.2%</td> 
   <td><strong>ARM64</strong></td> 
  </tr> 
  <tr> 
   <td>6144 MB</td> 
   <td>4</td> 
   <td>463 ms</td> 
   <td>429 ms</td> 
   <td>-7.9%</td> 
   <td>x86_64</td> 
  </tr> 
  <tr> 
   <td>8192 MB</td> 
   <td>5</td> 
   <td>338 ms</td> 
   <td>330 ms</td> 
   <td>-2.4%</td> 
   <td>x86_64</td> 
  </tr> 
  <tr> 
   <td>10240 MB</td> 
   <td>6</td> 
   <td><strong>280 ms</strong></td> 
   <td>292 ms</td> 
   <td>+4.1%</td> 
   <td><strong>ARM64</strong></td> 
  </tr> 
 </tbody> 
</table> 
<h3>Key Observations</h3> 
<p><strong>Cold Start Performance</strong>: Rust’s cold start initialization times are consistently between 19-28 ms across all memory configurations and architectures. ARM64 (<a href="https://aws.amazon.com/pm/ec2-graviton/">Graviton2</a>) shows slightly faster cold starts (19-23 ms) compared to x86_64 (26-29 ms). Both are significantly faster than interpreted runtimes because the binary is pre-compiled.</p> 
<p><strong>Near-Linear Scaling</strong>: Both architectures achieve impressive speedups:</p> 
<ul> 
 <li>ARM64: <strong>6.73x speedup</strong> with 6 workers (exceeds theoretical 6x!)</li> 
 <li>x86_64: 5.72x speedup with 6 workers</li> 
</ul> 
<p><strong>Latency Consistency</strong>: The P95 and P99 metrics show excellent consistency:</p> 
<ul> 
 <li>ARM64 at 6 vCPUs: P50=278ms, P95=292ms, P99=292ms (low variance)</li> 
 <li>x86_64 at 6 vCPUs: P50=292ms, P95=298ms, P99=298ms</li> 
</ul> 
<p>Both architectures show consistent latency at maximum parallelization.</p> 
<h2>Cost Analysis</h2> 
<p>Let’s analyze the cost implications of different configurations for processing 20 bcrypt hashes.</p> 
<p><strong>Cost Comparison: ARM64 vs x86_64</strong> (us-east-1, as of January 2026):</p> 
<table width="100%"> 
 <thead> 
  <tr> 
   <td>Config</td> 
   <td>Memory</td> 
   <td>Workers</td> 
   <td>ARM64 Duration</td> 
   <td>ARM64 Cost/1M</td> 
   <td>x86_64 Duration</td> 
   <td>x86_64 Cost/1M</td> 
   <td>Cheaper Arch</td> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>1 vCPU</td> 
   <td>1536 MB</td> 
   <td>1</td> 
   <td>1,885 ms</td> 
   <td>$38.60</td> 
   <td>1,671 ms</td> 
   <td>$42.78</td> 
   <td><strong>ARM64</strong></td> 
  </tr> 
  <tr> 
   <td><strong>2 vCPU</strong></td> 
   <td><strong>2048 MB</strong></td> 
   <td><strong>2</strong></td> 
   <td><strong>1,334 ms</strong></td> 
   <td><strong>$36.46</strong></td> 
   <td><strong>1,253 ms</strong></td> 
   <td><strong>$42.77</strong></td> 
   <td><strong>ARM64 *</strong></td> 
  </tr> 
  <tr> 
   <td>3 vCPU</td> 
   <td>4096 MB</td> 
   <td>3</td> 
   <td>685 ms</td> 
   <td>$37.47</td> 
   <td>892 ms</td> 
   <td>$60.80</td> 
   <td><strong>ARM64</strong></td> 
  </tr> 
  <tr> 
   <td>4 vCPU</td> 
   <td>6144 MB</td> 
   <td>4</td> 
   <td>463 ms</td> 
   <td>$37.97</td> 
   <td>429 ms</td> 
   <td>$44.00</td> 
   <td><strong>ARM64</strong></td> 
  </tr> 
  <tr> 
   <td>5 vCPU</td> 
   <td>8192 MB</td> 
   <td>5</td> 
   <td>338 ms</td> 
   <td>$36.94</td> 
   <td>330 ms</td> 
   <td>$45.10</td> 
   <td><strong>ARM64</strong></td> 
  </tr> 
  <tr> 
   <td>6 vCPU</td> 
   <td>10240 MB</td> 
   <td>6</td> 
   <td>280 ms</td> 
   <td>$38.27</td> 
   <td>292 ms</td> 
   <td>$49.87</td> 
   <td><strong>ARM64</strong></td> 
  </tr> 
 </tbody> 
</table> 
<h5>*Cheaper Arch</h5> 
<p><strong>Cost Formulas:</strong></p> 
<ul> 
 <li>ARM64: (Memory in GB) × (Duration in seconds) × $0.0000133334</li> 
 <li>x86_64: (Memory in GB) × (Duration in seconds) × $0.0000166667 (25% higher rate)</li> 
</ul> 
<p><strong>Key Insight</strong>: The <strong>2 vCPU ARM64 configuration provides the lowest cost</strong> at $36.46 per million invocations while achieving 1.41x speedup. All ARM64 configurations remain cost-competitive ($36-$39 range) despite significant performance differences, demonstrating how increased throughput can offset higher memory costs.</p> 
<p><strong>Choosing the Right Configuration</strong>:</p> 
<table width="100%"> 
 <thead> 
  <tr> 
   <td><strong>Priority</strong></td> 
   <td><strong>Recommended Config</strong></td> 
   <td><strong>Rationale</strong></td> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>Lowest Cost</td> 
   <td>ARM64, 2048 MB, 2 workers</td> 
   <td>$36.46/1M invocations, 1.41x speedup</td> 
  </tr> 
  <tr> 
   <td>Balanced</td> 
   <td>ARM64, 4096 MB, 3 workers</td> 
   <td>$37.47/1M invocations, 2.75x speedup</td> 
  </tr> 
  <tr> 
   <td>Low Latency</td> 
   <td>ARM64, 10240 MB, 6 workers</td> 
   <td>280ms avg, 6.73x speedup</td> 
  </tr> 
 </tbody> 
</table> 
<h2>When to Use Multi-threaded Rust on Lambda</h2> 
<h3>Recommended Use Cases</h3> 
<ul> 
 <li><strong>Batch data processing</strong>: Transform, validate, or enrich large datasets</li> 
 <li><strong>Cryptographic operations</strong>: Hashing, encryption, digital signatures</li> 
 <li><strong>Image/video processing</strong>: Resize, transcode, analyze media files</li> 
 <li><strong>Scientific computing</strong>: Simulations, data analysis, machine learning inference</li> 
 <li><strong>High-volume workloads</strong>: Functions invoked &gt;100,000 times per day benefit from optimization</li> 
</ul> 
<h3>When to Consider Alternatives</h3> 
<ul> 
 <li><strong>I/O-bound operations</strong>: Use async Rust instead of multi-threading for database queries or API calls</li> 
 <li><strong>Simple transformations</strong>: Functions completing in &lt;100ms rarely benefit from parallelization</li> 
 <li><strong>Low-volume workloads</strong>: Development overhead may not be justified for &lt;10,000 invocations per day</li> 
 <li><strong>Rapid prototyping</strong>: Python or Node.js may be more appropriate when iteration speed is critical</li> 
</ul> 
<h2>Cleanup</h2> 
<p>To delete the resources created in this post:</p> 
<pre><code class="language-bash"># Delete the Lambda function
aws lambda delete-function --function-name rust-multithread-lambda

# Delete the CloudWatch log group
aws logs delete-log-group --log-group-name /aws/lambda/rust-multithread-lambda</code></pre> 
<p><strong>Note</strong>: If you deployed multiple configurations for testing, you’ll need to delete each function individually by repeating the delete command with each function name, or use the SAM template for bulk cleanup:</p> 
<pre><code class="language-bash">aws cloudformation delete-stack --stack-name rust-multithread-benchmark</code></pre> 
<h2>Conclusion</h2> 
<p>When you allocate more memory to your Lambda function, AWS provides proportionally more vCPUs—up to 6 vCPUs at 10,240 MB. However, <strong>sequential code only uses one vCPU</strong>, leaving the additional compute power idle while you pay for the full allocation. Multi-threaded Rust with Rayon enables you to harness all available vCPUs for CPU-intensive workloads, transforming unused capacity into real performance gains.</p> 
<p>Our benchmarks demonstrate this clearly:</p> 
<ul> 
 <li><strong>Near-linear scaling</strong>: ARM64 achieved <strong>6.73x speedup</strong> with 6 workers—you get proportional returns on your vCPU investment</li> 
 <li><strong>Fast cold starts</strong>: 19-28 ms initialization across all configurations, eliminating the cold start concerns often associated with compiled languages</li> 
 <li><strong>Consistent latency</strong>: ARM64 at 6 vCPUs shows only 1ms variance between P50 and P99, critical for predictable response times</li> 
 <li><strong>Cost efficiency</strong>: ARM64 is 15-20% cheaper than x86_64 with better scaling at maximum parallelization</li> 
</ul> 
<p><strong>The key takeaway</strong>: If your Lambda function performs CPU-intensive work and you’re allocating more than 1,769 MB of memory, you likely have multiple vCPUs available. Without multi-threading, those vCPUs sit idle. Rayon’s parallel iterators allow you to switch from sequential to parallel processing by changing <code>.iter()</code> to <code>.par_iter()</code> in your code.</p> 
<p><strong>Recommended starting point</strong>: ARM64 with 4096 MB (3 workers) offers an excellent balance of cost and performance for most workloads. Scale up to 6 vCPUs for latency-critical applications, or down to 2 vCPUs for maximum cost savings.</p> 
<h2>Additional Resources</h2> 
<ul> 
 <li><a href="https://github.com/awslabs/aws-lambda-rust-runtime">AWS Lambda Rust Runtime</a></li> 
 <li><a href="https://www.cargo-lambda.info/">Cargo Lambda Documentation</a></li> 
 <li><a href="https://docs.rs/rayon/latest/rayon/">Rayon Data Parallelism Library</a></li> 
 <li><a href="https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html">AWS Lambda Memory and CPU Configuration</a></li> 
 <li><a href="https://aws.amazon.com/lambda/pricing/">AWS Lambda Pricing</a></li> 
</ul> 
<p><em>The complete sample code, SAM template, and test scripts from this post are available at </em><a href="https://github.com/aws-samples/sample-rust-multithread-lambda"><em>Github Repository</em></a><em>.</em></p>
