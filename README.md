# Artifact

This artifact will benchmark LFI and various WebAssembly engines on the SPEC
2017 suite. You must have a license for SPEC 2017 and already have access to
the file `cpu2017.iso`. If you don't have SPEC 2017, you will only be able to
run the microbenchmarks. The code for LFI is available at
https://github.com/zyedidia/lfi.

It must be run on a Linux ARM64 machine. On Apple Silicon you may run from
inside a virtual machine on MacOS or use Asahi Linux. Other possible ARM64
machines include Google Cloud's T2A instance, AWS Graviton 3, Raspberry Pi 4 or
5, Orange Pi 5, etc...

1. Install Podman.

```
sudo apt install podman
```

2. Download and import the Podman container.

```
wget https://github.com/zyedidia/lfi-artifact/releases/download/pre-built/lfi.tar.xz
podman import lfi.tar.xz lfi
```

3. Download SPEC 2017 and install it to `/home/$USER/cpu2017`.

If you have access to SPEC 2017:

Instructions [here](https://www.spec.org/cpu2017/Docs/install-guide-unix.html). For Linux, run:

```
mount -t iso9660 -o ro,exec,loop cpu2017.iso /mnt
cd /mnt
./install.sh
```


4. Enter the Podman container

If you don't have SPEC 2017, skip the `-v ~/cpu2017:/home/lfi/cpu2017:U` argument.

```
podman run -v ~/cpu2017:/home/lfi/cpu2017:U -it --user lfi --workdir /home/lfi --name lfi --security-opt=seccomp=unconfined lfi /bin/bash
```

# SPEC 2017 benchmark

Only run these commands if you have SPEC 2017:

1. Set up the benchmark

Inside the container:

```
./setup.sh
```

2. Check with the fast script (optional)

If you want to do a check before running the full benchmark, you can run
(inside the container):

```
./fast-run-and-report.sh
```

This should run relatively quickly (2 minutes) and produce plots in the
`~/cpu2017/stats` directory (same as the full benchmark, but the plots will be
incomplete and only use the `test` benchmark size).

3. Run the benchmark

Inside the container:

```
./run-and-report.sh
```

This step will take multiple hours (8 hours on an M1 Mac).

After this step completes, the results and PNG plots will be placed in
`~/cpu2017/stats` (accessible both from the host and from inside the
container).

These graphs should reproduce the results from Figures 3 and 4 (and Table 4,
which is just a summary of the geometric means).

# CoreMark (optional, if not using SPEC 2017)

If you don't have SPEC 2017, you can run the CoreMark benchmark. This benchmark
wasn't included in the results of the paper, but can serve as an open
benchmark.

```
cd coremark
./bench.sh
```

This should take a few minutes.

Our results on an M1 Mac running Asahi Linux were:

```
$ cat result/lfi-overhead.txt
1.10
$ cat result/wasm2c-overhead.txt
1.16
$ cat result/wamr-overhead.txt
1.13
$ cat result/wasmtime-overhead.txt
2.66
```

# Microbenchmarks

Inside the container:

```
cd microbenchmarks
make
./run-linux.sh
./run-lfi.sh
```

This should roughly reproduce the results from Table 5 and take less than 1
minute to run.

## gVisor (optional)

Running gVisor is optional. If your setup can support gVisor (4K pages), you
can run the benchmarks with gVisor as well. Unfortunately these benchmarks
cannot be run from inside Podman, so you must copy the directory to your host
and run the binaries there. You may need to wait 10x as long for the gVisor
benchmarks to complete, since these benchmarks are significantly slower with
gVisor than with Linux. Run these commands outside the container:

```
podman cp lfi:/home/lfi/microbenchmarks .
cd microbenchmarks
sudo ./gvisor/runsc --network none do /bin/bash
# ./run-linux.sh
```

# KVM vs LFI (optional)

This is more difficult to benchmark in a containerized manner since it involves
measuring the overhead of virtualization. This benchmark is also not a core
result from the paper, so we have it here as an optional extra. Run an ARM64
virtual machine, and run the container from inside the virtual machine.

Inside the container in the VM:

```
# make sure setup.sh has been run
./bench-invm.sh
```

Then, inside a container not running in a virtualized environment:

```
# make sure setup.sh has been run
./bench-outvm.sh
```

You'll need to manually use the `specstats` program (from this repository) to
compare the results from the two containers.

We used Vagrant and qemu-kvm to run this benchmark. Make sure they are both
installed, then run `vagrant up` and `vagrant ssh` with the provided
`Vagrantfile`.
