# LeDoBE

## Ledger Live Dockerized Build Environment

Build [Ledger Live][ledger] components in an isolated Docker environment with persistence between sessions.

A GNU Make `Makefile` with a number of [targets] is included to simplify building some of the Ledger components, but be aware that the supported component range is very limited at the moment. Truth to be told, this entire project began with the goal of building just the Ledger Live [desktop application][tg-desktop] for a Raspberry Pi 4, which is an ARM64 device that did not get any official support from [@LedgerHQ][ledgerhq] at the time.

### Supported platforms

At this point only ARM64 Linux is supported out of the box.

With some tinkering you should be able to make it work for other platforms. Consider submitting a pull request, if you manage to also come up with a good workflow for multiple platforms.


## Installation

### Requirements

- Docker (tested with v18.09.1+)

#### Recommended

- A POSIX compliant shell


### Process

Once you have the LeDoBE source code (that is available at <https://github.com/antichris/ledobe>), either

- run

	```sh
	bin/build
	```

- or build an image from the included `docker` directory:

	```sh
	docker build -t ledobe docker
	```

	You can replace `ledobe` with a tag of your choosing, just be consistent in all invocations from here on.


## Operation

### TL;DR

```sh
bin/run make
```

If you find that your builds are taking a flippin' forever, have a look-see at the [section on jobs][jobs].


### Details

#### Starting Containers

The `bin/run` creates directories to persist your build data and spins up and attaches to a container. You can also do this manually:

```sh
mkdir home build
docker run -it --rm \
	-v"$(readlink -f home):/mnt/ledobe" \
	-v"$(readlink -f build):/build" \
	ledobe
```

#### Building stuff

Containers run bash shell by default, which is where you would execute

```sh
make
```

to start building Ledger Live components. A bunch of make [targets] have recipes to make it way easier to build them, see the relevant section below.


#### Jobs

Concurrent execution can speed up things tremendously. If you have sufficient memory (both physical and swap), you should increase `make` jobs up to the max threads that your device can handle. On a Raspberry Pi 4B that would be 4:

```sh
make -j4
```

If you don't have enough memory, but you decide to run with increased jobs anyway, expect crashes as the kernel OOM killer kicks in.

Here is an article on [increasing the swap size on a Raspberry Pi][rpi-swap].


## `make` Targets

[targets]: #make-targets (`make` Targets)

By default `make` is going to build the Ledger Live desktop application, but you can specify it a different target to build, e.g.:

```sh
make coreNPM
```

The following sections contain a brief overview of the explicit targets currently supported by the included `Makefile`.

1. ### `desktop`

	[tg-desktop]: #desktop (Target `desktop`)

	The Ledger Live desktop application. ([GitHub][gh-desktop])

	This is likely the one you are most interested in building. It having no official binaries for ARM64 (and ARM in general) served as the primary motivation for this project.


2. ### `coreLib`

	Ledger Core Library used by Ledger applications. ([GitHub][gh-core])


3. ### `coreNPM`

	Ledger Core Library cross-platform C++ bindings for NodeJS. ([GitHub][gh-coreNPM])


## License

The source code of this project is released under [Mozilla Public License Version 2.0][mpl]. See [LICENSE](LICENSE).

[ledger]: https://www.ledger.com/ledger-live
	"Ledger Live: Most trusted & secure crypto wallet | Ledger"
[ledgerhq]: https://github.com/LedgerHQ
	"Ledger"

[jobs]: #jobs
	"Operation ❭ Details ❭ Jobs"

[gh-desktop]: https://github.com/LedgerHQ/ledger-live-desktop
	"LedgerHQ/ledger-live-desktop: Ledger Live (Desktop) - GitHub"
[gh-coreNPM]: https://github.com/LedgerHQ/lib-ledger-core-node-bindings
	"LedgerHQ/lib-ledger-core-node-bindings - GitHub"
[gh-core]: https://github.com/LedgerHQ/lib-ledger-core
	"LedgerHQ/lib-ledger-core - GitHub"

[rpi-swap]: https://nebl.io/neblio-university/enabling-increasing-raspberry-pi-swap/
	"Enabling & Increasing Raspberry Pi Swap - Neblio"

[mpl]: https://www.mozilla.org/en-US/MPL/2.0/
	"Mozilla Public License, version 2.0"
