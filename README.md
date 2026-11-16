# Gamma Pulse

A C++17 market-making engine for BitMEX, with WebSocket market data, REST order entry, and configurable two-sided quoting. The CMake project and runtime configuration retain the original `GAMMA_RAY` / `gamma-ray` names.

## Features

- **Three quoting strategies:** `Top`, `Mid`, and `AvellanedaStoikov`.
- **Quote ladders:** configurable levels, price spacing, and size increments.
- **Inventory-aware pricing:** position-based skew and an inventory-dependent reservation price in the Avellaneda–Stoikov strategy.
- **Exchange integration:** market data, order and position updates, batch order creation/amendment/cancellation, and rate-limit monitoring.
- **Research tools:** historical-data scripts, sample datasets, and a strategy-calibration notebook.

## Requirements

The existing build targets a Linux/POSIX environment. On Windows, use a Linux environment such as WSL; the source includes `unistd.h` and the build uses GCC/Clang-style flags.

Install the following development tools and libraries:

| Dependency | Requirement |
| --- | --- |
| C++ compiler | C++17 support |
| CMake | 3.18 or newer |
| Ninja | Used by the provided build scripts |
| Boost | 1.65 or newer, with System and Thread |
| POCO | 1.10 or newer, with Foundation, Util, and Net |
| C++ REST SDK | `cpprestsdk`, with its CMake package |
| OpenSSL | SSL and Crypto libraries and headers |
| Git submodules | doctest and WebSocket++ |
| Doxygen | Optional, for API documentation |

**Build paths:** `CMakeLists.txt` currently expects POCO headers under `/usr/local/include/Poco`. `cmake/Cpprestsdk.cmake` also sets a Linux-specific package path. Adjust these paths if your dependencies are installed elsewhere. The shell scripts set `LD_LIBRARY_PATH=/usr/local/lib` when launching executables.

## Build

Run these commands from the repository root in a POSIX shell.

Initialize the dependencies. The submodule configuration uses `git://` URLs; the command below rewrites them to HTTPS for this invocation:

```sh
git -c url.https://github.com/.insteadOf=git://github.com/ submodule update --init --recursive
```

Configure and compile a debug build:

```sh
cmake -S . -B debug -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build debug
```

For a release build:

```sh
cmake -S . -B release -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build release
```

The executables are `main` and `unit_tests` in the selected build directory. In-source builds are disabled.

## Configure and run

The application reads **`gamma-ray.json` from its current working directory**. It has no command-line configuration argument. For the debug build:

```sh
cp sample-config.json debug/gamma-ray.json
```

Edit `debug/gamma-ray.json` before starting:

- Replace `apiKey` and `apiSecret` with your own credentials. Do not rely on the values committed in the sample.
- Keep `websocketUrl` and `httpUrl` pointed at the intended environment. The sample uses BitMEX testnet.
- Review `contract`, quote sizes, and strategy parameters.

The executable submits and amends orders once connected; there is no dry-run switch. Start with testnet credentials. Keep your local configuration private: `gamma-ray.json` is not explicitly ignored by `.gitignore`, although the `debug/` and `release/` directories are ignored.

Launch from the build directory so the configuration can be found:

```sh
cd debug
LD_LIBRARY_PATH=/usr/local/lib ./main
```

For release, copy the configuration into `release/gamma-ray.json` and run from `release/` instead.

The convenience scripts build and then launch their respective executable from the build directory:

```sh
sh run-debug.sh
sh run-release.sh
sh run-test.sh
```

Run each script from the repository root. The application scripts require the configuration file in their corresponding build directory.

## Quoting configuration

See [`sample-config.json`](sample-config.json) for the complete configuration structure. All three strategy parameter blocks are read at startup, even when only one strategy is selected, so keep them present.

| Field under `quotingParam` | Meaning |
| --- | --- |
| `quotingMode` | `Top`, `Mid`, or `AvellanedaStoikov` |
| `size` | Base order quantity on each side |
| `pairs` | Number of bid/ask levels; use a positive integer |
| `priceInterval` | Price spacing between successive levels |
| `sizeIncrement` | Additional quantity at each successive level |
| `skewFactor` | Inventory skew applied in tick units per base-size position increment; `0` disables skew |
| `quotingModeParam.Top.spread` | Minimum spread around fair value when improving the best bid/ask |
| `quotingModeParam.Mid.spread` | Spread centered on fair value |
| `quotingModeParam.AvellanedaStoikov.gamma` | Risk-aversion parameter |
| `quotingModeParam.AvellanedaStoikov.sigma` | Volatility parameter |
| `quotingModeParam.AvellanedaStoikov.k` | Order-arrival parameter |

The Avellaneda–Stoikov implementation uses a rolling 24-hour horizon starting when the strategy is constructed. Parameters are loaded from configuration rather than calibrated automatically.

The sample also contains `targetBasePosition`, `positionOffset`, `tradesPerMinute`, and `tradeRateSeconds`. These are loaded into the parameter model, but should not be treated as enforced position or trading-rate limits. Target-position handling is marked as a TODO in the dispatcher, and its margin-check call is commented out. Exchange instrument details are hardcoded in `src/bitmex_gateway.cpp`; review them before selecting another contract.

## Tests

After building, run the registered test executable through CTest:

```sh
ctest --test-dir debug --output-on-failure
```

Alternatively, use `sh run-test.sh` to build and launch `unit_tests`.

Currently, `tests/CMakeLists.txt` includes only the doctest runner, `tests/main.cpp`; every module test file is commented out. A successful invocation therefore does not establish coverage of those modules. Enable the desired files in `TESTFILES` and rebuild to run them. Some exchange tests connect to BitMEX and submit or cancel orders; inspect their endpoints and credentials before enabling them.

## Documentation and research

If Doxygen is available at configure time, generate the API documentation with:

```sh
cmake --build debug --target doc
```

The generated HTML entry point is `debug/html/index.html`.

The [`doc/`](doc/) directory contains:

- `01_fetch_bitmex_data.py`: downloads historical quote and trade archives; its default range is approximately one year.
- `02_extract_and_concat.py`: reads the downloaded archives with pandas and writes per-symbol CSV files.
- `AS model calibration.ipynb`: an Avellaneda–Stoikov calibration notebook.
- `data/`: sample XBTUSD quote and trade data.

Run the Python scripts from `doc/` because their data paths are relative to the working directory. The extraction script requires pandas; Python/Jupyter dependencies are separate from the C++ build.

## Repository layout

```text
app/          Application entry point and component wiring
src/          Exchange adapters, pricing, quoting, and order dispatch
include/      Headers, interfaces, models, and bundled JSON library
tests/        doctest runner and module/integration test sources
cmake/        Dependency discovery and build helpers
external/     doctest and WebSocket++ submodules
doc/          Calibration notebook, data scripts, and samples
```

## License

Released under the [ISC license](LICENSE). Copyright (c) 2020 Liangyuan Tian.
