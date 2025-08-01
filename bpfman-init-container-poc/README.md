# bpfman Init Container Proof of Concept

## Overview

This PoC demonstrates using bpfman as an init container in Kubernetes to manage eBPF programs. The solution uses a sidecar pattern where:

- **Init Container (`bpfman-init`)**: Loads and attaches eBPF programs (XDP counter) to the network interface via bpfman, then runs continuously as a sidecar
- **Main Container**: Runs an application that consumes data from the eBPF program via shared BPF maps
- **Lifecycle Management**: The init container handles cleanup on pod termination via preStop hooks

The PoC specifically loads the go-xdp-counter example program that counts network packets and attaches it to the `eth0` interface, while the main container runs a Go application to read and display the statistics.

## Architecture

```
┌─────────────────┐    ┌─────────────────┐
│  Init Container │    │  Main Container │
│   (bpfman-init) │    │                 │
│                 │    │                 │
│ • Load eBPF     │    │ • Read BPF maps │
│ • Attach XDP    │    │ • Display stats │
│ • Cleanup on    │    │                 │
│   termination   │    │                 │
└─────────────────┘    └─────────────────┘
        │                      │
     (mounts)               (mounts)
        │                      │
    /run/bpfman      /sys/fs/bpf/xdp_stats_map
    /var/lib/bpfman
    /sys/fs/bpf
```

## Prerequisites

* Tested on the bpfman example cluster (make run-on-kind).
* podman

## How to Run

Steps 1. and 2. can be skipped if the defaults are used. The image already exists at quay.io/akaris/bpfman-init-container:latest

### 1. Build the Container Image (Optional)

```bash
make build # IMAGE=example.com/foo/bar:latest
```

This builds the image using the `Containerfile.bpfman.initcontainer` which will include:
- 1) bpfman binary
- 2) Example eBPF bytecode including go-xdp-counter
- 3) Example eBPF golang applications including go-xdp-counter
> Note: This is all in one image for the purpose of this PoC. The main container only needs  3).
> The init contaienr only 1) and 2), or if the bytecode was loaded via an image then only 1).

### 2. Push the Image (Optional)

```bash
make push # IMAGE=example.com/foo/bar:latest
```

### 3. Deploy to Kubernetes

```bash
make deploy # NAMESPACE=change-me-or-default-bpftest
```

This will:
- Create a `bpftest` namespace with privileged security settings
- Deploy the pod with both init and main containers and volume mounts for BPF

### 4. Check Pod Status and Logs

```bash
make logs
```
> You will have to wait until all containers are done initializing.

## Verification

Run `make logs` to see the output from both containers:

### Expected Output

```
# make logs
kubectl get pods bpfman-init-container-test
NAME                         READY   STATUS    RESTARTS   AGE
bpfman-init-container-test   2/2     Running   0          72s

bpfman-init
================
kubectl logs -c bpfman-init bpfman-init-container-test
++ get_ids
++ bpfman list programs --application XDPCounter
++ awk '/XDPCounter/ {print $1}'
+ bpfman load file -p /usr/local/lib/ebpf/go-xdp-counter/bpf_x86_bpfel.o --programs xdp:xdp_stats --application XDPCounter
 Bpfman State                                                      
 BPF Function:  xdp_stats                                          
 Program Type:  xdp                                                
 Path:          /usr/local/lib/ebpf/go-xdp-counter/bpf_x86_bpfel.o 
 Global:        None                                               
 Metadata:      bpfman_application=XDPCounter                      
 Map Pin Path:  /run/bpfman/fs/maps/6321                           
 Map Owner ID:  None                                               
 Maps Used By:  6321                                               
 Links:         None                                               

 Kernel State                                               
 Program ID:                       6321                     
 BPF Function:                     xdp_stats                
 Kernel Type:                      xdp                      
 Loaded At:                        2025-08-01T11:56:07+0000 
 Tag:                              4d23a1d7f3618653         
 GPL Compatible:                   true                     
 Map IDs:                          [46]                     
 BTF ID:                           6760                     
 Size Translated (bytes):          232                      
 JITted:                           true                     
 Size JITted:                      140                      
 Kernel Allocated Memory (bytes):  4096                     
 Verified Instruction Count:       21                       

++ get_ids
++ tail
++ bpfman list programs --application XDPCounter
++ awk '/XDPCounter/ {print $1}'
+ ID=6321
+ bpfman attach 6321 xdp --iface eth0 --priority 100
 Bpfman State                                      
 BPF Function:       xdp_stats                     
 Program Type:       xdp                           
 Program ID:         6321                          
 Link ID:            4130150334                    
 Interface:          eth0                          
 Priority:           100                           
 Position:           0                             
 Proceed On:         pass, dispatcher_return       
 Network Namespace:  None                          
 Metadata:           bpfman_application=XDPCounter 

+ sleep infinity

main-container
================
kubectl logs -c main-container bpfman-init-container-test
+ /usr/local/lib/ebpf/go-xdp-counter/go-xdp-counter --crd
2025/08/01 11:56:18 285 packets received
2025/08/01 11:56:18 55680 bytes received

2025/08/01 11:56:21 285 packets received
2025/08/01 11:56:21 55680 bytes received
```

## Cleanup

```bash
make undeploy
```

This removes the pod and deletes the test namespace.
