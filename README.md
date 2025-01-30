# TNIC sw evaluation

We next explain how to build and run in our Intel cluster with the following hw (Intel(R) Core(TM) i9-9900K (8 cores, 3.2 GHz), Intel Corporation Ethernet Controllers (XL710)) and OS (NixOS, 5.15.4). 

Please note that specific changes and configurations are for this particular cluster.

You need to clone and fetch the following software dependencies:


- eRPC (`39b0fefb0cecc7845a3b8e82a3a2ed339a975891`)
- dpdk (`dpdk-stable-19.11.14`)


## Intel cluster

### System setup
We pin port-1 to DPDK-driver (igb_uio) in all servers we want to use. Note, it should only be port-1 in this cluster as we have the NFS mounted on port-0!

The script can be found in the DPDK codebase which you can get online as explain next.
 
```sh
sudo python dpdk-devbind.py --bind=igb_uio 0000:01:00.1/enp1s0f1
```

You can find the correct interfaces to bind with 

```sh
sudo python dpdk-devbind.py --status 
```


### Build dpdk 

We get the DPDK 19.11.14 (LTS) from https://core.dpdk.org/download/ 

Comment out the app and kernel dirs from GNUMakefile and execute

```sh
make install T=x86_64-native-linuxapp-gcc DESTDIR=usr CXFLAGS="-Wno-error"
```


### Build eRPC library

```sh
git clone https://github.com/erpc-io/eRPC.git	
```


Modify the CMakefile.txt by adding this compile option: `-msse4.1` and removing the `-Werror`

Include the local paths with dpdk-installation if DPDK is not globally installed, etc. to the CMakefile.txt

Change this line in the following file: `eRPC/src/rpc_impl/rpc_rx.cc:line 22`

```sh
if (unlikely(!pkthdr->check_magic())) {
	ERPC_WARN("Rpc %u: Received packet %s with bad magic number.Dropping.\n", 
	rpc_id, pkthdr->to_string().c_str());
	continue;
}	
```


Then we build eRPC as a library 

```sh 
mkdir build && cd build && cmake .. -DPERF=OFF -DTRANSPORT=dpdk
cp src/config.h ../src
make
```

In our systems, we could only properly link the applications with the following linking flags that need to be added to any Makefile

```sh 
-Wl,--whole-archive -ldpdk -Wl,--no-whole-archive -lrte_ethdev -Wl,-lrte_port
```


### How to run

Install system dependencies and get a nix-shell.

```sh
nix-shell -p
```

The prototypes of the applications are in the following folders: `bft-pb`, `bft-cr`, `trusted-log` and `a2m-single-node`.

In each of the folders there exists a ```CMakeLists.txt``` file. You can build each application (which will build all required binaries) with 

```
mkdir build && cd build && cmake .. && make -j
```

Prior you try build the application, you need to update the ```common.h``` file with your configuration as well as the relative paths of the installed dependencies in the ```CMakeLists.txt``` file.

The network benchmarking is in the `network_bench` folder with instructions.

The various systems that can provide attestations (Intel-SGX based SSL-server and AMD-SEV-based SSL server) can be build following the instructions in `sha256-example/digital_signatures/`. You need to run the `server` and `client` binaries. 

Please activate the body of `process()` function in the `attest_server.cpp` if you want to include also the overhead of the computation. We used the `bench` binary for microbenchmarking in our servers.