# Getting Started

The [wiki](https://github.com/betrusted-io/betrusted-wiki/wiki) is going to be the most up-to-date source of information for getting started, as it is still a topic in flux.

Below are some excerpts from the Wiki, but some links may be out of date.

## Update Your Device
* [Updating](https://github.com/betrusted-io/betrusted-wiki/wiki/Updating-Your-Device) your device.
* Videos: [Install a debug cable](https://vimeo.com/676414220/a590a017c3); [Assemble a Limited Edition](https://vimeo.com/676415520/51f9df8439)
* [Bleeding-edge binaries](https://ci.betrusted.io/latest-ci/) can be found at the CI server. Use at your own risk.
* [Releases](https://ci.betrusted.io/releases/): please check the corresponding README within each subdir for notes.

## Setting up Security
* [Inspecting your mainboard](https://github.com/betrusted-io/betrusted-wiki/wiki/Inspecting-Your-Mainboard#trusted-domain-point-by-point)
* [Initialize Root Keys](https://github.com/betrusted-io/betrusted-wiki/wiki/Initializing-Root-Keys) on "factory new" devices, by selecting the item from the main menu.
* (Optional) [Burn Battery-Backed RAM keys](https://github.com/betrusted-io/betrusted-wiki/wiki/FAQ:-FPGA-AES-Encryption-Key-(eFuse-BBRAM)#how-do-i-externally-provision-my-device) Note: do not use this if you plan to store long-term secrets on the device.

## Jargon
* [Jargon](https://github.com/betrusted-io/betrusted-wiki/wiki/Jargon): Confused by terms like SoC and EC? You're not alone.

## Other Issues
### Pre-Boot & Security
* [What happens before boot?](https://github.com/betrusted-io/betrusted-wiki/wiki/How-Does-Precursor-Get-to-the-Reset-Vector%3F) fills in the details of everything that happens before the first instruction gets run.
* "[Secure Boot](https://github.com/betrusted-io/betrusted-wiki/wiki/Secure-Boot-and-KEYROM-Layout)" and key ROM layout
* [eFuse/BBRAM FPGA key](https://github.com/betrusted-io/betrusted-wiki/wiki/FAQ:-FPGA-AES-Encryption-Key-(eFuse-BBRAM)) FAQ

### Between the Software and Hardware: Hardware Abstractions
* [UTRA](https://github.com/betrusted-io/xous-core/tree/main/svd2utra) Hardware register access abstraction for Xous
* [Peripheral access conventions](https://github.com/betrusted-io/xous-core/wiki/Peripheral-Access-Conventions) Goals for hardware register abstractions
* [COM](https://github.com/betrusted-io/betrusted-wiki/wiki/Embedded-Controller-(EC)-COM-Protocol) Protocol between the embedded controller (EC) and the main SoC

### Hardware Documentation
* [SoC register set](https://ci.betrusted.io/betrusted-soc/doc/index.html) / generated from [SoC Litex Design Source](https://github.com/betrusted-io/betrusted-soc/blob/main/betrusted_soc.py)
* [SoC block diagram](https://github.com/betrusted-io/betrusted-soc#readme) is embedded in the README for the SoC
* [EC register set](https://ci.betrusted.io/betrusted-ec/doc/) / generated from [EC Litex Design Source](https://github.com/betrusted-io/betrusted-ec/blob/main/betrusted_ec.py)
* [Hardware design files](https://github.com/betrusted-io/betrusted-hardware) PDFs of schematics are in the "mainboard-*" directories

### TRNG Chronicles
* [Physics and electrical design](https://betrusted.io/avalanche-noise) of the external Avalanche generator
* Notes on [Characterization](https://github.com/betrusted-io/betrusted-wiki/wiki/TRNG-characterization) and debugging of raw sources
* [On-line health monitoring](https://github.com/betrusted-io/betrusted-wiki/wiki/TRNG-Online-Health-Monitors)
* [Post-Generation Conditioning](https://github.com/betrusted-io/betrusted-wiki/wiki/TRNG-Data-Conditioning) with ChaCha

### Audit Trail
* [crate-scraper](https://github.com/betrusted-io/crate-scraper) is the beginning of a tool that helps with audit trails. It saves all the source code derived from `crates.io` to build Xous, and collates all the `build.rs` files into a [single mega-file](https://github.com/betrusted-io/crate-scraper/blob/main/builds.rs) for faster manual inspection.

### Meta-Issues
* [imports](https://github.com/betrusted-io/xous-core/tree/main/imports) How imported repositories that are not yet stand-alone crates are managed
* [Emulation](https://github.com/betrusted-io/xous-core/tree/main/emulation)
* [Tools](https://github.com/betrusted-io/xous-core/tree/main/tools) Helper tools to build bootable images
* [Converting Wiki Pages](https://github.com/betrusted-io/betrusted-wiki/wiki/Going-from-ODT-to-Github-Wiki) from ODT to Markdown
