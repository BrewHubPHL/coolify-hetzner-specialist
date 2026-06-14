# Chapter 11: Docker and Wasm

## Core Idea

Wasm is the third wave of cloud workloads (after VMs and containers)—smaller, faster, and more portable for specific use cases—and Docker Desktop already supports building, pushing, and running Wasm apps as OCI images inside minimal scratch containers via pluggable Wasm runtimes (Spin, wasmtime, etc.).

## Frameworks Introduced

- **Three waves of cloud computing**: VMs → Linux containers → Wasm; each wave pushes smaller, faster, more secure workloads; all three coexist.
  - When to use: Choosing workload type—Wasm for serverless functions, plugins, edge, some AI; containers for general-purpose apps with heavy I/O and complex networking.
  - How: Compile to Wasm (e.g. `wasm32-wasip1`); package with `FROM scratch` + `--platform wasi/wasm`; run with `--runtime=io.containerd.spin.v2`.

- **Wasm container pattern**: Wasm binary + minimal metadata in a scratch-based OCI image, executed by a containerd Wasm runtime—not a full Linux userspace.
  - When to use: Shipping Wasm apps through Docker Hub and running with familiar `docker build` / `docker run`.
  - How: Dockerfile copies `.wasm` and runtime config (`spin.toml`); build with `--platform wasi/wasm`; run with explicit `--runtime` and `--platform=wasi/wasm`.

- **Spin framework workflow**: Scaffold → edit → `spin build` → test with `spin up` → containerize → push → `docker run`.
  - When to use: Rapid Rust (or other) Wasm HTTP services on Fermyon Spin.
  - How: `spin new hello-world -t http-rust`; `rustup target add wasm32-wasip1`; `spin build` produces `hello_world.wasm`.

## Key Concepts

- **Wasm / WebAssembly**: Portable VM target; apps compile to Wasm instead of OS/arch-specific binaries.
- **Wasm runtime**: Execution engine (Spin, wasmtime, wasmer, wasmedge, slight, lunatic, wws) registered with containerd.
- **Wasm container**: Wasm binary running inside scratch image; managed like any container but without Linux OS.
- **wasi/wasm platform**: OCI platform identifier for Wasm images (`docker build --platform wasi/wasm`).
- **scratch base image**: Empty image—Wasm containers need no Linux distribution.
- **spin.toml**: Spin app manifest; `source` path must match where Dockerfile places the `.wasm` file.

## Mental Models

- **Wasm complements containers, not replaces them**: Containers are more flexible for networking and I/O; Wasm wins on size (~104 kB images vs MB+) and cold start for narrow workloads.
- **Think "compile once, run on any Wasm runtime"**: Portability at the Wasm layer; Docker adds registry and ops familiarity.
- **Runtime flag selects the Wasm engine**: `io.containerd.spin.v2` for Spin apps; Docker Desktop bundles multiple runtimes.
- **Path alignment in spin.toml matters**: Dockerfile `COPY` destination must match `source` in `spin.toml`.

## Anti-patterns

- **Using Wasm for heavy I/O or complex networking**: Current ecosystem limitations; prefer Linux containers.
- **Forgetting to update spin.toml after COPY path change**: Build succeeds but runtime cannot find the Wasm binary.
- **Omitting `--platform wasi/wasm` on build and run**: Image won't be recognized or executed as Wasm.
- **Expecting vulnerability scanning on Wasm images**: Scanners cannot analyze Wasm images yet (as of book edition).
- **Using Multipass Docker VMs for Wasm**: Beta feature requires Docker Desktop 4.37+ with Wasm enabled.

## Code Examples

```bash
rustup target add wasm32-wasip1
spin --version
spin new hello-world -t http-rust
spin build
spin up
```

- **What it demonstrates**: Local Spin dev loop before containerization.

```rust
Ok(http::Response::builder()
    .status(200)
    .header("content-type", "text/plain")
    .body("Docker loves Wasm")?
    .build())
```

- **What it demonstrates**: Minimal Spin HTTP handler returning plain text.

```dockerfile
FROM scratch
COPY /target/wasm32-wasip1/release/hello_world.wasm .
COPY spin.toml .
```

```toml
[component.hello-world]
source = "hello_world.wasm"
```

```bash
docker build \
  --platform wasi/wasm \
  --provenance=false \
  -t nigelpoulton/ddd-book:wasm .

docker push nigelpoulton/ddd-book:wasm

docker run -d --name wasm-ctr \
  --runtime=io.containerd.spin.v2 \
  --platform=wasi/wasm \
  -p 5556:80 \
  nigelpoulton/ddd-book:wasm /
```

- **What it demonstrates**: Scratch-based Wasm image build, registry push, and Spin runtime container run mapped to host port 5556.

```bash
docker run --rm -i --privileged --pid=host \
  jorgeprendes420/docker-desktop-shim-manager:latest
```

- **What it demonstrates**: List installed containerd Wasm runtimes (wasmtime, spin, wasmer, wasmedge, etc.).

## Reference Tables

| Prerequisite | Version / setting |
|--------------|-------------------|
| Docker Desktop | 4.37+ with Wasm enabled |
| Rust | 1.82+ with `wasm32-wasip1` target |
| Spin | 3.1+ |
| Docker Desktop General | Use containerd for pulling/storing images |
| Docker Desktop Features | Enable Wasm |

| Wasm runtime (containerd plugin) | Example use |
|----------------------------------|---------------|
| `io.containerd.spin.v2` | Fermyon Spin apps (book examples) |
| `io.containerd.wasmtime.v1` | Wasmtime |
| `io.containerd.wasmer.v1` | Wasmer |
| `io.containerd.wasmedge.v1` | WasmEdge |
| `io.containerd.slight.v1` | Slight |
| `io.containerd.lunatic.v1` | Lunatic |
| `io.containerd.wws.v1` | Wasm Workers Server |

| Workload fit | Wasm | Linux container |
|--------------|------|-----------------|
| Serverless / edge functions | Strong | Possible |
| AI workloads (some) | Growing | Strong |
| Heavy I/O | Weak | Strong |
| Complex networking | Weak | Strong |
| Image size | ~KB | MB+ |

## Worked Example

Build and run `hello-world` Spin app as a Wasm container:

1. Docker Desktop: enable containerd image store + Wasm (Features in development).
2. `spin new hello-world -t http-rust` → edit `src/lib.rs` body to `"Docker loves Wasm"`.
3. `spin build` → produces `target/wasm32-wasip1/release/hello_world.wasm`.
4. `spin up` → verify at `http://127.0.0.1:3000/hello`; Ctrl-C to stop.
5. Create Dockerfile (`FROM scratch`, COPY wasm + spin.toml); fix `spin.toml` `source = "hello_world.wasm"`.
6. `docker build --platform wasi/wasm --provenance=false -t <user>/ddd-book:wasm .` — image ~104 kB.
7. `docker push <user>/ddd-book:wasm` — Hub shows `wasi/wasm` platform.
8. `docker run -d --name wasm-ctr --runtime=io.containerd.spin.v2 --platform=wasi/wasm -p 5556:80 <user>/ddd-book:wasm /`
9. Browse `http://localhost:5556/hello` — same response inside Wasm container.
10. `docker rm wasm-ctr -f && docker rmi <user>/ddd-book:wasm`

## Key Takeaways

1. Wasm apps compile to a portable VM target; Docker packages them as tiny scratch-based OCI images.
2. Always use `--platform wasi/wasm` for build and run; specify `--runtime` matching your framework (Spin → `io.containerd.spin.v2`).
3. Docker Desktop ships multiple Wasm runtimes via containerd plugins.
4. Wasm excels at small, fast, secure workloads—not yet at heavy I/O or complex networking.
5. Align `spin.toml` source paths with Dockerfile COPY destinations.
6. Wasm images push/pull to standard OCI registries; vulnerability scanning not yet supported.

## Connects To

- **Ch 10 (Docker and AI)**: Wasm modules listed alongside models as OCI artifact types in unified registries.
- **Ch 5–7 (images/containers)**: Same `docker build`, `docker push`, `docker run` toolchain with platform/runtime flags.
- **Cloudflare Workers / edge**: Wasm's natural deployment surface; Docker Desktop previews local Wasm container workflow.
- **Serverless patterns**: Spin + Wasm containers mirror function-as-container deployment models.
