# Renode Emulation

[Renode](https://renode.io/) is a multi-device emulator written in C#. It is designed to assist in testing and development of software, and is also useful in developing new hardware blocks.

The emulator is available for Windows, Mac, and Linux. It is designed to simulate whole systems of devices, meaning it can easily capture the interactions between devices on a network or bus. It allows you to pause the system and inspect memory, single-step, and watch various sections of the bus.

There is extensive end-user documentation available at [Read the Docs](https://renode.readthedocs.io/en/latest/), which is highly recommended. The remainder of this chapter will cover recommendations on how to use Renode with Xous.

## Quickstart using the Renode emulator

Xous uses [Renode](https://renode.io/) as the preferred emulator, because it is easy to extend the hardware peripherals without recompiling the entire emulator.

[Download Renode](https://renode.io/#downloads) and ensure it is in your path.
For now, you need to [download the nightly build](https://dl.antmicro.com/projects/renode/builds/),
until `DecodedOperation` is included in the release.

Then, build Xous:

```sh
cargo xtask renode-image
```

This will compile everything in `release` mode for RISC-V, compile the tools required to package it all up, then create an image file.

Finally, run Renode and specify the `xous-release.resc` REnode SCript:

```sh
renode emulation/xous-release.resc
```

Renode will start emulation automatically, and will run the same set of programs as in "Hosted mode".

### Network support

If you are seeking to emulate a Xous build with apps requiring network support, then there is some additional setup.

Renode accepts the Ethernet frames that the EC sends to the simulated WF200, and then dumps them onto a network TAP interface called `renodetap`.

It is probably best to prepare the TAP interface and associated configuration before the emulation:
* **linux distro with Network Manager:** `nmcli c add con-name renodetun type tun ifname renodetun mode tap ipv4.method shared ipv6.method ignore`
* **OpenSuSE Aeon:** also requires installation of dnsmasq `transactional-update pkg install dnsmasq`

Start the emulation with:

```sh
renode emulation/xous-release-tap.resc
```
Once the emulation is up and running then connect to one of the WiFi APs Renode1 ... Renode9 (any password will work)
