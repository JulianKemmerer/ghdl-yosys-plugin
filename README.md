<p align="center">
  <a title="GHDL synthesis documentation" href="https://ghdl.github.io/ghdl/using/Synthesis.html"><img src="https://img.shields.io/website.svg?label=ghdl.github.io%2Fghdl&longCache=true&style=flat-square&url=http%3A%2F%2Fghdl.github.io%2Fghdl%2Findex.html"></a><!--
  -->
  <a title="Join the chat at https://gitter.im/ghdl1/Lobby" href="https://gitter.im/ghdl1/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge"><img src="https://img.shields.io/badge/chat-on%20gitter-4db797.svg?longCache=true&style=flat-square&logo=gitter&logoColor=e8ecef"></a><!--
  -->
  <a title="Docker Images" href="https://github.com/ghdl/docker"><img src="https://img.shields.io/docker/pulls/ghdl/synth.svg?logo=docker&logoColor=e8ecef&style=flat-square&label=docker"></a><!--
  -->
  <a title="'push' workflow Status" href="https://github.com/ghdl/ghdl-yosys-plugin/actions?query=workflow%3Apush"><img alt="'push' workflow Status" src="https://img.shields.io/github/workflow/status/ghdl/ghdl-yosys-plugin/push?longCache=true&style=flat-square&label=push&logo=Github%20Actions&logoColor=fff"></a>
</p>

# ghdl-yosys-plugin: VHDL synthesis (based on [ghdl](https://github.com/ghdl/ghdl) and [yosys](https://github.com/YosysHQ/yosys))

**This is experimental and work in progress!** See [ghdl.rtfd.io: Using/Synthesis](http://ghdl.readthedocs.io/en/latest/using/Synthesis.html).

- [Build as a module (shared library)](#build-as-a-module-shared-library)
- [Usage](#Usage)
- [Docker](#Docker)

> TODO: Create table with features of VHDL that are supported, WIP and pending.

---

## Build as a module (shared library)

- Get and install [yosys](https://github.com/YosysHQ/yosys).
- Get sources, build and install [ghdl](https://github.com/ghdl/ghdl). Ensure that GHDL is configured with synthesis features (enabled by default since v0.37). See [Building GHDL](https://github.com/ghdl/ghdl#building-ghdl).

> NOTE: GHDL must be built with at least version of 8 GNAT (`gnat-8`).

> HINT: The default build prefix is `/usr/local`. Sudo permission might be required to install tools there.

- Get and build ghdl-yosys-plugin: `make`.

> HINT: If ghdl is not available in the PATH, set `GHDL` explicitly, e.g.: `make GHDL=/my/path/to/ghdl`.

The output is a shared library (`ghdl.so` on GNU/Linux), which can be used directly: `yosys -m ghdl.so`.

To install the module, the library must be copied to `YOSYS_PREFIX/share/yosys/plugins/ghdl.so`, where `YOSYS_PREFIX` is the installation path of yosys. This can be achieved through a make target: `make install`.

Alternatively, the shared library can be copied/installed along with ghdl:

```sh
cp ghdl.so "$GHDL_PREFIX/lib/ghdl_yosys.so"

yosys-config --exec mkdir -p --datdir/plugins
yosys-config --exec ln -s "$GHDL_PREFIX/lib/ghdl_yosys.so" --datdir/plugins/ghdl.so
```

## Usage

Example for icestick, using ghdl, yosys, nextpnr and icestorm:

```sh
cd examples/icestick/leds/

# Analyse VHDL sources
ghdl -a leds.vhdl
ghdl -a spin1.vhdl

# Synthesize the design.
# NOTE: if ghdl is built as a module, set MODULE to '-m ghdl' or '-m path/to/ghdl.so'
yosys $MODULE -p 'ghdl leds; synth_ice40 -json leds.json'

# P&R
nextpnr-ice40 --package hx1k --pcf leds.pcf --asc leds.asc --json leds.json

# Generate bitstream
icepack leds.asc leds.bin

# Program FPGA
iceprog leds.bin
```

Alternatively, it is possible to analyze, elaborate and synthesize VHDL sources at once, instead of calling ghdl and yosys in two steps. In this example:

```
yosys $MODULE -p 'ghdl leds.vhdl spin1.vhdl -e leds; synth_ice40 -json leds.json'
```

## Docker

Docker image [`ghdl/synth:beta`](https://hub.docker.com/r/ghdl/synth/tags) includes yosys and the ghdl module (shared library). These can be used to synthesize designs straightaway. For example:

```sh
docker run --rm -t \
  -v $(pwd)/examples/icestick/leds:/src \
  -w /src \
  ghdl/synth:beta \
  yosys -m ghdl -p 'ghdl leds.vhdl blink.vhdl -e leds; synth_ice40 -json leds.json'
```

> In a system with [docker](https://docs.docker.com/install) installed, the image is automatically downloaded the first time invoked.

Furthermore, the snippet above can be extended in order to P&R the design with [nextpnr](https://github.com/YosysHQ/nextpnr) and generate a bitstream with [icestorm](https://github.com/cliffordwolf/icestorm) tools:

```sh
cd examples/icestick/leds/

DOCKER_CMD="$(command -v winpty) docker run --rm -it -v /$(pwd)://wrk -w //wrk"

$DOCKER_CMD ghdl/synth:beta     yosys -m ghdl -p 'ghdl leds.vhdl rotate4.vhdl -e leds; synth_ice40 -json leds.json'
$DOCKER_CMD ghdl/synth:nextpnr  nextpnr-ice40 --hx1k --json leds.json --pcf leds.pcf --asc leds.asc
$DOCKER_CMD ghdl/synth:icestorm icepack leds.asc leds.bin

iceprog leds.bin
```

> NOTE: on GNU/Linux, it should be possible to use `iceprog` through `ghdl/synth:icestorm`. On Windows and macOS, accessing USB/COM ports of the host from containers seems not to be supported yet. Therefore, `iceprog` is required to be available on the host.

## Build as part of yosys (not recommended)

- Get and build ghdl as in the previous section.

- Get [yosys](https://github.com/YosysHQ/yosys) sources.

- Get ghdl-yosys-plugin and:
  - Patch yosys sources using `yosys.diff`.
  - Copy `src/*` to `yosys/frontends/ghdl`.
  - Configure yosys by adding (to) `Makefile.conf`:

```makefile
ENABLE_GHDL := 1
GHDL_DIR := <ghdl install dir>
```

- Build and install yosys.
