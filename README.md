# mobilitydb-duckdb-bridge

Generated DuckDB loadable extension that bridges the **mobilitydb** DataFission wasm shim into DuckDB as native scalar functions, aggregates, UDTFs, custom column types, and identity casts.

Produced by [`ducklink-shim-codegen`](https://github.com/zacharywhitley/ducklink-shim-codegen) from a shim-interface SQLite database. **Do not edit by hand** — regenerate from the source.

## Surface

| Extension | Version | Scalars | Aggregates | UDTFs | Windows | Types | Operators | Casts | Preprocessors | Catalog | Indexes |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| mobilitydb | 0.1.0 | 1548 | 54 | 43 | 0 | 31 | 32 | 36 | 1 | 0 | 3 |

## Build

```sh
cargo build --release
```

The build needs sibling checkouts of the path-dep'd workspace
crates (`datafission-df-plugin-loader`, `datafission-df-plugin-api`,
`datafission-functions`) at `../datafission/crates/`.

## Package as a loadable extension

DuckDB only loads files ending in `.duckdb_extension` that carry
a metadata footer matching the target DuckDB version. The
[`extension-template-rs`](https://github.com/duckdb/extension-template-rs)
project ships a helper script for this:

```sh
python3 path/to/append_extension_metadata.py \
  -l target/release/libmobilitydb_duckdb_bridge.dylib \
  -n mobilitydb_duckdb_bridge \
  -p osx_arm64 \
  -dv v1.2.0 \
  -ev v0.1.0 \
  -o mobilitydb_duckdb_bridge.duckdb_extension \
  --abi-type C_STRUCT
```

Adjust `-p` to your platform (`linux_amd64`, `linux_arm64`,
`osx_amd64`, …). Pin `-dv` to the DuckDB version you target.

## Load + use

The bridge needs the composed shim wasm at runtime; set
`MOBILITYDB_SHIM_WASM` before `LOAD`:

```sql
-- Need DuckDB CLI with -unsigned since we don't sign the bridge.
MOBILITYDB_SHIM_WASM=/path/to/mobilitydb-composed.wasm duckdb -unsigned
> LOAD '/path/to/mobilitydb_duckdb_bridge.duckdb_extension';
```

## Regen

When the upstream shim's SQL surface changes:

```sh
cd ~/git/ducklink-shim-codegen
cargo run --release -- \
  --interface /path/to/mobilitydb-interface.sqlite \
  --out ~/git/mobilitydb-duckdb-bridge
```

The codegen pipes every emitted `.rs` through
`rustfmt --edition 2021`, so the resulting crate is
`cargo fmt -p mobilitydb-duckdb-bridge -- --check`-clean by
construction.

## Architecture

- Scalars: dispatched through duckdb-rs's `VScalar` trait
  (vectorised; one chunk per invoke).
- Aggregates + custom types + identity casts: raw
  `libduckdb-sys` FFI — duckdb-rs doesn't surface these in its
  safe wrapper.
- Entrypoint: hand-rolled `extern "C"` symbol named
  `mobilitydb_duckdb_bridge_init_c_api` (NOT the
  `#[duckdb_entrypoint_c_api]` macro) so we can pull both a raw
  `duckdb_connection` AND a `Connection` wrapper.
- Custom types like `GEOMETRY` / `STBOX` register as BLOB
  aliases; identity casts in both directions let
  `CREATE TABLE t (g GEOMETRY); INSERT INTO t VALUES
  (st_geomfromtext(...))` round-trip.

## License

Apache-2.0. Generated source so the same license as the
codegen.
