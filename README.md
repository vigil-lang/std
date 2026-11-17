# The Vigil standard library

This is the standard library of [Vigil](https://vigil-lang.github.io), a
statically typed language that compiles to eBPF. It is written in Vigil. It
provides:
- the contexts and kernel helpers of each kind of program;
- packet headers;
- fixed-point arithmetic;
- small machine-learning models;
- probabilistic sketches.

Each file is a module: `net.vg` is `std.net`, and `ml/tensor.vg` is
`std.ml.tensor`.

```python
# Drops IPv4 packets from sources that sent more than 1000 of them.
from std.net import XdpContext, XdpAction, EthHdr, Ipv4Hdr, ETH_HLEN, ETH_P_IP
from std.sketch import CountMin

map counts: PerCpuArray[CountMin[4, 1024], 1]

@xdp
def heavy_hitters(ctx: XdpContext) -> XdpAction:
    eth = ctx.parse[EthHdr](0)?
    if eth.proto != ETH_P_IP:
        return XdpAction.PASS
    ip = ctx.parse[Ipv4Hdr](ETH_HLEN)?
    sketch = counts[0]?
    if sketch.add_estimate(ip.saddr) > 1000:
        return XdpAction.DROP
    return XdpAction.PASS
```

## Modules

| Module | Contents |
|---|---|
| `std.bpf` | tracing programs: `KprobeContext`, `TracepointContext`, `LsmContext`, and helpers (`pid_tgid`, `uid_gid`, `ktime_ns`, `comm`, `current_task`, `read_user_str`, `read_kernel_str`, `prandom_u32`, ...) |
| `std.net` | packet programs: `XdpContext`, `TcContext`, `XdpAction`, `TcAction`; the `@network_order` headers `EthHdr`, `Ipv4Hdr`, `Ipv6Hdr`, `TcpHdr`, `UdpHdr`, `Byte`; constants (`ETH_P_IP`, `IPPROTO_TCP`, ...) |
| `std.fixed` | `fixed[I, F]`, saturating fixed-point numbers with I integer bits and F fractional bits, and `q8_8`, `q16_16`, `q4_12` |
| `std.ml` | the `Classifier` protocol and its models: `Linear`, `Mlp`, `Perceptron`, `Centroids`, `DecisionTree`, `RandomForest`, `Knn` |
| `std.ml.tensor` | `Vector[T, N]` and `Matrix[T, R, C]` with static shapes; `argmax`, `relu` |
| `std.ml.features` | per-packet stream statistics for flow classifiers: `Welford`, `Log2Histogram`, `FlowFeatures` |
| `std.sketch` | `CountMin`, `Bloom` and `HyperLogLog`, sized at compile time |
| `std.testing` | packet descriptions (`Packet`, `Damaged`, `Trace`, `Mix`) for the tests of programs |
| `std.debug` | tools for testing the toolchain itself |

Each module starts with a comment explaining its conventions. Some examples:
- how `ctx.parse` checks bounds;
- why `fixed` saturates instead of wrapping;
- why shapes are generic arguments.

A few comments mention `vigil test` and `vigil verify`. They belong to the
full Vigil, not to the simplified compiler.

## How the compiler uses it

The compiler (`vigil-lang/vigil`) embeds this library in its binary when
it is built. It reads the library from `../std`, so the two repositories
are cloned side by side:

```sh
git clone https://github.com/vigil-lang/vigil compiler
git clone https://github.com/vigil-lang/std std
cd compiler && cargo build --release
```

The file `std.ref` of the compiler names the branch or tag of this
repository that its CI and releases build with.

A project can use another copy without rebuilding the compiler. It does
this through a dependency named `std` in its `vigil.toml`:

```toml
[dependencies]
std = { path = "../std" }                               # a local directory
# std = { github = "vigil-lang/std", tag = "v0.1.0" }   # or a version of this repository
```

Such a dependency replaces the built-in library entirely, so it must have
every module the programs import.

## Contributing

Code in this repository runs inside the kernel and must pass its verifier:
- loops have constant bounds;
- packet accesses go through `ctx.parse`;
- there is no floating point (use `std.fixed`);
- values must fit the 512-byte stack: large state belongs in maps.

The library may use a few privileged features that user code may not, such
as `@context` and `@helper` declarations.

To check a change, build and test the compiler against it:

```sh
cd ../compiler
cargo test --workspace         # checks that every module is formatted, builds the examples
cargo run -p vigil-cli -- fmt ../std
```

`.github/workflows/ci.yml` does the same on every push and pull request,
with the compiler of `main`.

## Versions

Releases are tags (`v0.1.0`). A release of the compiler names one in its
`std.ref`, so it can be rebuilt with exactly the same library.
