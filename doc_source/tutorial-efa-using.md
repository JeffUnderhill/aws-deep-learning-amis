# Using EFA on the DLAMI<a name="tutorial-efa-using"></a>

The following section describes how to use EFA to run multi\-node applications on the AWS Deep Learning AMI\.

**Topics**
+ [Running Multi\-Node Applications with EFA](#tutorial-efa-using-multi-node)

## Running Multi\-Node Applications with EFA<a name="tutorial-efa-using-multi-node"></a>

To run an application across a cluster of nodes some configuration is needed\.

### Enable Passwordless SSH<a name="tutorial-efa-using-multi-node-ssh"></a>

Select one node in your cluster as the leader node\. The remaining nodes are referred to as the member nodes\.  

1. On the leader node, generate the RSA keypair\.

   ```
   ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
   ```

1. Change the permissions of the private key on the leader node\.

   ```
   chmod 600 ~/.ssh/id_rsa
   ```

1. Copy the public key `~/.ssh/id_rsa.pub` to and append it to `~/.ssh/authorized_keys` of the member nodes in the cluster\. 

1. You should now be able to directly login to the member nodes from the leader node using the private ip\.

   ```
   ssh <member private ip>
   ```

1. Disable strictHostKeyChecking and enable agent forwarding on the leader node by adding the following to the \~/\.ssh/config file on the leader node: 

   ```
   Host *
       ForwardAgent yes
   Host *
       StrictHostKeyChecking no
   ```

1. On Amazon Linux and Amazon Linux 2 instances, run the following command on the leader node to provide correct permissions to the config file:

   ```
   chmod 600 ~/.ssh/config
   ```

### Create Hosts File<a name="tutorial-efa-using-multi-node-hosts"></a>

On the leader node, create a hosts file to identify the nodes in the cluster\. The hosts file must have an entry for each node in the cluster\. Create a file \~/hosts and add each node using the private ip as follows:  

```
localhost slots=8
<private ip of node 1> slots=8
<private ip of node 2> slots=8
```

### 2\-Node NCCL Plugin Check<a name="tutorial-efa-using-2node"></a>

The nccl\_message\_transfer is a simple test to ensure that the NCCL OFI Plugin is working as expected\. The test validates functionality of NCCL's connection establishment and data transfer APIs\. Make sure you use the complete path to mpirun as shown in the example while running NCCL applications with EFA\. Change the params `np` and `N` based on the number of instances and GPUs in your cluster\. For more information, see the [AWS OFI NCCL documentation](https://github.com/aws/aws-ofi-nccl/tree/master/tests)\.

#### P3dn\.24xlarge check<a name="tutorial-efa-using-2node-P3dn.24xlarge"></a>

The following nccl\_message\_transfer test is for CUDA 10\.0\. You can run the commands for CUDA 10\.1 and 10\.2 by replacing the CUDA version\.

```
$/opt/amazon/openmpi/bin/mpirun \
         -n 2 -N 1 --hostfile hosts \
         -x LD_LIBRARY_PATH=/usr/local/cuda-10.0/efa/lib:/usr/local/cuda-10.0/lib:/usr/local/cuda-10.0/lib64:/usr/local/cuda-10.0:$LD_LIBRARY_PATH \
         -x FI_PROVIDER="efa" --mca btl tcp,self --mca btl_tcp_if_exclude lo,docker0 --bind-to none \
         ~/src/bin/efa-tests/efa-cuda-10.0/nccl_message_transfer
```

Your output should look like the following\. You can check the output to see that EFA is being used as the OFI provider\.

```
INFO: Function: ofi_init Line: 702: NET/OFI Forcing AWS OFI ndev 4
INFO: Function: ofi_init Line: 714: NET/OFI Selected Provider is efa
INFO: Function: main Line: 49: NET/OFI Process rank 1 started. NCCLNet device used on ip-172-31-15-30 is AWS Libfabric.
INFO: Function: main Line: 53: NET/OFI Received 4 network devices
INFO: Function: main Line: 57: NET/OFI Server: Listening on dev 0
INFO: Function: ofi_init Line: 702: NET/OFI Forcing AWS OFI ndev 4
INFO: Function: ofi_init Line: 714: NET/OFI Selected Provider is efa
INFO: Function: main Line: 49: NET/OFI Process rank 0 started. NCCLNet device used on ip-172-31-15-30 is AWS Libfabric.
INFO: Function: main Line: 53: NET/OFI Received 4 network devices
INFO: Function: main Line: 57: NET/OFI Server: Listening on dev 0
INFO: Function: main Line: 96: NET/OFI Send connection request to rank 0
INFO: Function: main Line: 69: NET/OFI Send connection request to rank 1
INFO: Function: main Line: 100: NET/OFI Server: Start accepting requests
INFO: Function: main Line: 73: NET/OFI Server: Start accepting requests
INFO: Function: main Line: 103: NET/OFI Successfully accepted connection from rank 0
INFO: Function: main Line: 107: NET/OFI Rank 1 posting 255 receive buffers
INFO: Function: main Line: 76: NET/OFI Successfully accepted connection from rank 1
INFO: Function: main Line: 80: NET/OFI Sent 255 requests to rank 1
INFO: Function: main Line: 131: NET/OFI Got completions for 255 requests for rank 0
INFO: Function: main Line: 131: NET/OFI Got completions for 255 requests for rank 1
```

##### Multi\-node NCCL Performance Test on P3dn\.24xlarge<a name="tutorial-efa-using-multi-node-performance"></a>

To check NCCL Performance with EFA, run the standard NCCL Performance test that is available on the official [NCCL\-Tests Repo](https://github.com/NVIDIA/nccl-tests.git)\. The DLAMI comes with this test already built for both CUDA 10\.0, 10\.1, and 10\.2\. You can similarly run your own script with EFA\. 

When constructing your own script, refer to the following guidance:
+ Provide the FI\_PROVIDER="efa" flag to enable EFA use\.
+ Use the complete path to mpirun as shown in the example while running NCCL applications with EFA\.
+ Change the params np and N based on the number of instances and GPUs in your cluster\.
+ Add the NCCL\_DEBUG=INFO flag and make sure that the logs indicate EFA usage as "Selected Provider is EFA"\.

Use the command `watch nvidia-smi` on any of the member nodes to monitor GPU usage\. The following `watch nvidia-smi` commands are for CUDA 10\.0 and depend on the Operating System of your instance\. You can run the commands for CUDA 10\.1 and 10\.2 by replacing the CUDA version\.
+ Amazon Linux and Amazon Linux 2:

  ```
  $ /opt/amazon/openmpi/bin/mpirun \
           -x FI_PROVIDER="efa" -n 16 -N 8 \
           -x NCCL_DEBUG=INFO \
           -x NCCL_TREE_THRESHOLD=0 \
           -x LD_LIBRARY_PATH=/usr/local/cuda-10.0/efa/lib:/usr/local/cuda-10.0/lib:/usr/local/cuda-10.0/lib64:/usr/local/cuda-10.0:/opt/amazon/efa/lib64:/opt/amazon/openmpi/lib64:$LD_LIBRARY_PATH \
           --hostfile hosts --mca btl tcp,self --mca btl_tcp_if_exclude lo,docker0 --bind-to none \
           $HOME/src/bin/efa-tests/efa-cuda-10.0/all_reduce_perf -b 8 -e 1G -f 2 -g 1 -c 1 -n 100
  ```
+ Ubuntu 16\.04 and Ubuntu 18\.04:

  ```
  $ /opt/amazon/openmpi/bin/mpirun \
           -x FI_PROVIDER="efa" -n 16 -N 8 \
           -x NCCL_DEBUG=INFO \
           -x NCCL_TREE_THRESHOLD=0 \
           -x LD_LIBRARY_PATH=/usr/local/cuda-10.0/efa/lib:/usr/local/cuda-10.0/lib:/usr/local/cuda-10.0/lib64:/usr/local/cuda-10.0:/opt/amazon/efa/lib:/opt/amazon/openmpi/lib:$LD_LIBRARY_PATH \
           --hostfile hosts --mca btl tcp,self --mca btl_tcp_if_exclude lo,docker0 --bind-to none \
           $HOME/src/bin/efa-tests/efa-cuda-10.0/all_reduce_perf -b 8 -e 1G -f 2 -g 1 -c 1 -n 100
  ```

Your output should look like the following\.

```
# nThread 1 nGpus 1 minBytes 8 maxBytes 1073741824 step: 2(factor) warmup iters: 5 iters: 100 validation: 1
#
# Using devices
#   Rank  0 Pid   3801 on ip-172-31-41-105 device  0 [0x00] Tesla V100-SXM2-32GB
#   Rank  1 Pid   3802 on ip-172-31-41-105 device  1 [0x00] Tesla V100-SXM2-32GB
#   Rank  2 Pid   3803 on ip-172-31-41-105 device  2 [0x00] Tesla V100-SXM2-32GB
#   Rank  3 Pid   3804 on ip-172-31-41-105 device  3 [0x00] Tesla V100-SXM2-32GB
#   Rank  4 Pid   3805 on ip-172-31-41-105 device  4 [0x00] Tesla V100-SXM2-32GB
#   Rank  5 Pid   3807 on ip-172-31-41-105 device  5 [0x00] Tesla V100-SXM2-32GB
#   Rank  6 Pid   3810 on ip-172-31-41-105 device  6 [0x00] Tesla V100-SXM2-32GB
#   Rank  7 Pid   3813 on ip-172-31-41-105 device  7 [0x00] Tesla V100-SXM2-32GB
#   Rank  8 Pid   4124 on ip-172-31-41-36 device  0 [0x00] Tesla V100-SXM2-32GB
#   Rank  9 Pid   4125 on ip-172-31-41-36 device  1 [0x00] Tesla V100-SXM2-32GB
#   Rank 10 Pid   4126 on ip-172-31-41-36 device  2 [0x00] Tesla V100-SXM2-32GB
#   Rank 11 Pid   4127 on ip-172-31-41-36 device  3 [0x00] Tesla V100-SXM2-32GB
#   Rank 12 Pid   4128 on ip-172-31-41-36 device  4 [0x00] Tesla V100-SXM2-32GB
#   Rank 13 Pid   4130 on ip-172-31-41-36 device  5 [0x00] Tesla V100-SXM2-32GB
#   Rank 14 Pid   4132 on ip-172-31-41-36 device  6 [0x00] Tesla V100-SXM2-32GB
#   Rank 15 Pid   4134 on ip-172-31-41-36 device  7 [0x00] Tesla V100-SXM2-32GB
ip-172-31-41-105:3801:3801 [0] NCCL INFO Bootstrap : Using [0]ens5:172.31.41.105<0>
ip-172-31-41-105:3801:3801 [0] NCCL INFO NET/OFI Forcing AWS OFI ndev 4
ip-172-31-41-105:3801:3801 [0] NCCL INFO NET/OFI Selected Provider is efa
NCCL version 2.4.8+cuda10.0
ip-172-31-41-105:3810:3810 [6] NCCL INFO Bootstrap : Using [0]ens5:172.31.41.105<0>
ip-172-31-41-105:3810:3810 [6] NCCL INFO NET/OFI Forcing AWS OFI ndev 4
ip-172-31-41-105:3810:3810 [6] NCCL INFO NET/OFI Selected Provider is efa
ip-172-31-41-105:3805:3805 [4] NCCL INFO Bootstrap : Using [0]ens5:172.31.41.105<0>
ip-172-31-41-105:3805:3805 [4] NCCL INFO NET/OFI Forcing AWS OFI ndev 4
ip-172-31-41-105:3805:3805 [4] NCCL INFO NET/OFI Selected Provider is efa
ip-172-31-41-105:3807:3807 [5] NCCL INFO Bootstrap : Using [0]ens5:172.31.41.105<0>
ip-172-31-41-105:3807:3807 [5] NCCL INFO NET/OFI Forcing AWS OFI ndev 4
ip-172-31-41-105:3807:3807 [5] NCCL INFO NET/OFI Selected Provider is efa
ip-172-31-41-105:3803:3803 [2] NCCL INFO Bootstrap : Using [0]ens5:172.31.41.105<0>
ip-172-31-41-105:3803:3803 [2] NCCL INFO NET/OFI Forcing AWS OFI ndev 4
ip-172-31-41-105:3803:3803 [2] NCCL INFO NET/OFI Selected Provider is efa
ip-172-31-41-105:3813:3813 [7] NCCL INFO Bootstrap : Using [0]ens5:172.31.41.105<0>
ip-172-31-41-105:3802:3802 [1] NCCL INFO Bootstrap : Using [0]ens5:172.31.41.105<0>
ip-172-31-41-105:3813:3813 [7] NCCL INFO NET/OFI Forcing AWS OFI ndev 4
ip-172-31-41-105:3813:3813 [7] NCCL INFO NET/OFI Selected Provider is efa
ip-172-31-41-105:3802:3802 [1] NCCL INFO NET/OFI Forcing AWS OFI ndev 4
ip-172-31-41-105:3802:3802 [1] NCCL INFO NET/OFI Selected Provider is efa
ip-172-31-41-105:3804:3804 [3] NCCL INFO Bootstrap : Using [0]ens5:172.31.41.105<0>
ip-172-31-41-105:3804:3804 [3] NCCL INFO NET/OFI Forcing AWS OFI ndev 4
ip-172-31-41-105:3804:3804 [3] NCCL INFO NET/OFI Selected Provider is efa
ip-172-31-41-105:3801:3862 [0] NCCL INFO Setting affinity for GPU 0 to ffffffff,ffffffff,ffffffff
ip-172-31-41-105:3801:3862 [0] NCCL INFO NCCL_TREE_THRESHOLD set by environment to 0.
ip-172-31-41-36:4128:4128 [4] NCCL INFO Bootstrap : Using [0]ens5:172.31.41.36<0>
ip-172-31-41-36:4128:4128 [4] NCCL INFO NET/OFI Forcing AWS OFI ndev 4
ip-172-31-41-36:4128:4128 [4] NCCL INFO NET/OFI Selected Provider is efa

-----------------------------some output truncated-----------------------------------

ip-172-31-41-105:3804:3869 [3] NCCL INFO comm 0x7f8c5c0025b0 rank 3 nranks 16 cudaDev 3 nvmlDev 3 - Init COMPLETE
#
#                                                     out-of-place                       in-place
#       size         count    type   redop     time   algbw   busbw  error     time   algbw   busbw  error
#        (B)    (elements)                     (us)  (GB/s)  (GB/s)            (us)  (GB/s)  (GB/s)
ip-172-31-41-105:3801:3801 [0] NCCL INFO Launch mode Parallel
ip-172-31-41-36:4124:4191 [0] NCCL INFO comm 0x7f28400025b0 rank 8 nranks 16 cudaDev 0 nvmlDev 0 - Init COMPLETE
ip-172-31-41-36:4126:4192 [2] NCCL INFO comm 0x7f62240025b0 rank 10 nranks 16 cudaDev 2 nvmlDev 2 - Init COMPLETE
ip-172-31-41-105:3802:3867 [1] NCCL INFO comm 0x7f5ff00025b0 rank 1 nranks 16 cudaDev 1 nvmlDev 1 - Init COMPLETE
ip-172-31-41-36:4132:4193 [6] NCCL INFO comm 0x7ffa0c0025b0 rank 14 nranks 16 cudaDev 6 nvmlDev 6 - Init COMPLETE
ip-172-31-41-105:3803:3866 [2] NCCL INFO comm 0x7fe9600025b0 rank 2 nranks 16 cudaDev 2 nvmlDev 2 - Init COMPLETE
ip-172-31-41-36:4127:4188 [3] NCCL INFO comm 0x7f6ad00025b0 rank 11 nranks 16 cudaDev 3 nvmlDev 3 - Init COMPLETE
ip-172-31-41-105:3813:3868 [7] NCCL INFO comm 0x7f341c0025b0 rank 7 nranks 16 cudaDev 7 nvmlDev 7 - Init COMPLETE
ip-172-31-41-105:3810:3864 [6] NCCL INFO comm 0x7f5f980025b0 rank 6 nranks 16 cudaDev 6 nvmlDev 6 - Init COMPLETE
ip-172-31-41-36:4128:4187 [4] NCCL INFO comm 0x7f234c0025b0 rank 12 nranks 16 cudaDev 4 nvmlDev 4 - Init COMPLETE
ip-172-31-41-36:4125:4194 [1] NCCL INFO comm 0x7f2ca00025b0 rank 9 nranks 16 cudaDev 1 nvmlDev 1 - Init COMPLETE
ip-172-31-41-36:4134:4190 [7] NCCL INFO comm 0x7f0ca40025b0 rank 15 nranks 16 cudaDev 7 nvmlDev 7 - Init COMPLETE
ip-172-31-41-105:3807:3865 [5] NCCL INFO comm 0x7f3b280025b0 rank 5 nranks 16 cudaDev 5 nvmlDev 5 - Init COMPLETE
ip-172-31-41-36:4130:4189 [5] NCCL INFO comm 0x7f62080025b0 rank 13 nranks 16 cudaDev 5 nvmlDev 5 - Init COMPLETE
ip-172-31-41-105:3805:3863 [4] NCCL INFO comm 0x7fec100025b0 rank 4 nranks 16 cudaDev 4 nvmlDev 4 - Init COMPLETE
           8             2   float     sum    145.4    0.00    0.00  2e-07    152.8    0.00    0.00  1e-07
          16             4   float     sum    137.3    0.00    0.00  1e-07    137.2    0.00    0.00  1e-07
          32             8   float     sum    137.0    0.00    0.00  1e-07    137.4    0.00    0.00  1e-07
          64            16   float     sum    137.7    0.00    0.00  1e-07    137.5    0.00    0.00  1e-07
         128            32   float     sum    136.2    0.00    0.00  1e-07    135.3    0.00    0.00  1e-07
         256            64   float     sum    136.4    0.00    0.00  1e-07    137.4    0.00    0.00  1e-07
         512           128   float     sum    135.5    0.00    0.01  1e-07    151.0    0.00    0.01  1e-07
        1024           256   float     sum    151.0    0.01    0.01  2e-07    137.7    0.01    0.01  2e-07
        2048           512   float     sum    138.1    0.01    0.03  5e-07    138.1    0.01    0.03  5e-07
        4096          1024   float     sum    140.5    0.03    0.05  5e-07    140.3    0.03    0.05  5e-07
        8192          2048   float     sum    144.6    0.06    0.11  5e-07    144.7    0.06    0.11  5e-07
       16384          4096   float     sum    149.4    0.11    0.21  5e-07    149.3    0.11    0.21  5e-07
       32768          8192   float     sum    156.7    0.21    0.39  5e-07    183.9    0.18    0.33  5e-07
       65536         16384   float     sum    167.7    0.39    0.73  5e-07    183.6    0.36    0.67  5e-07
      131072         32768   float     sum    193.8    0.68    1.27  5e-07    193.0    0.68    1.27  5e-07
      262144         65536   float     sum    243.9    1.07    2.02  5e-07    258.3    1.02    1.90  5e-07
      524288        131072   float     sum    309.0    1.70    3.18  5e-07    309.0    1.70    3.18  5e-07
     1048576        262144   float     sum    709.3    1.48    2.77  5e-07    693.2    1.51    2.84  5e-07
     2097152        524288   float     sum   1116.4    1.88    3.52  5e-07   1105.7    1.90    3.56  5e-07
     4194304       1048576   float     sum   2088.9    2.01    3.76  5e-07   2157.3    1.94    3.65  5e-07
     8388608       2097152   float     sum   2869.7    2.92    5.48  5e-07   2847.2    2.95    5.52  5e-07
    16777216       4194304   float     sum   4631.7    3.62    6.79  5e-07   4643.9    3.61    6.77  5e-07
    33554432       8388608   float     sum   8769.2    3.83    7.17  5e-07   8743.5    3.84    7.20  5e-07
    67108864      16777216   float     sum    16964    3.96    7.42  5e-07    16846    3.98    7.47  5e-07
   134217728      33554432   float     sum    33403    4.02    7.53  5e-07    33058    4.06    7.61  5e-07
   268435456      67108864   float     sum    59045    4.55    8.52  5e-07    58625    4.58    8.59  5e-07
   536870912     134217728   float     sum   115842    4.63    8.69  5e-07   115590    4.64    8.71  5e-07
  1073741824     268435456   float     sum   228178    4.71    8.82  5e-07   224997    4.77    8.95  5e-07
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 2.80613
#
```

#### P4d\.24xlarge check<a name="tutorial-efa-using-2node-P4d.24xlarge"></a>

The following nccl\_message\_transfer test is for CUDA 11\.0\.

```
$/opt/amazon/openmpi/bin/mpirun \
         -n 2 -N 1 --hostfile hosts \
         -x LD_LIBRARY_PATH=/usr/local/cuda-11.0/efa/lib:/usr/local/cuda-11.0/lib:/usr/local/cuda-11.0/lib64:/usr/local/cuda-11.0:$LD_LIBRARY_PATH \
         -x FI_EFA_USE_DEVICE_RDMA=1 -x NCCL_ALGO=ring -x --mca pml ^cm \
         -x FI_PROVIDER="efa" --mca btl tcp,self --mca btl_tcp_if_exclude lo,docker0 --bind-to none \
         ~/src/bin/efa-tests/efa-cuda-11.0/nccl_message_transfer
```

Your output should look like the following\. You can check the output to see that EFA is being used as the OFI provider\.

```
INFO: Function: ofi_init Line: 1078: NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
INFO: Function: ofi_init Line: 1078: NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
INFO: Function: ofi_init Line: 1152: NET/OFI Selected Provider is efa
INFO: Function: main Line: 68: NET/OFI Process rank 1 started. NCCLNet device used on ip-172-31-79-191 is AWS Libfabric.
INFO: Function: main Line: 72: NET/OFI Received 4 network devices
INFO: Function: main Line: 107: NET/OFI Network supports communication using CUDA buffers. Dev: 3
INFO: Function: main Line: 113: NET/OFI Server: Listening on dev 3
INFO: Function: ofi_init Line: 1152: NET/OFI Selected Provider is efa
INFO: Function: main Line: 68: NET/OFI Process rank 0 started. NCCLNet device used on ip-172-31-70-99 is AWS Libfabric.
INFO: Function: main Line: 72: NET/OFI Received 4 network devices
INFO: Function: main Line: 107: NET/OFI Network supports communication using CUDA buffers. Dev: 3
INFO: Function: main Line: 113: NET/OFI Server: Listening on dev 3
INFO: Function: main Line: 126: NET/OFI Send connection request to rank 1
INFO: Function: main Line: 160: NET/OFI Send connection request to rank 0
INFO: Function: main Line: 164: NET/OFI Server: Start accepting requests
INFO: Function: main Line: 167: NET/OFI Successfully accepted connection from rank 0
INFO: Function: main Line: 171: NET/OFI Rank 1 posting 255 receive buffers
INFO: Function: main Line: 130: NET/OFI Server: Start accepting requests
INFO: Function: main Line: 133: NET/OFI Successfully accepted connection from rank 1
INFO: Function: main Line: 137: NET/OFI Sent 255 requests to rank 1
INFO: Function: main Line: 223: NET/OFI Got completions for 255 requests for rank 1
INFO: Function: main Line: 223: NET/OFI Got completions for 255 requests for rank 0
```

##### Multi\-node NCCL Performance Test on P4d\.24xlarge<a name="tutorial-efa-using-multi-node-performance"></a>

To check NCCL Performance with EFA, run the standard NCCL Performance test that is available on the official [NCCL\-Tests Repo](https://github.com/NVIDIA/nccl-tests.git)\. The DLAMI comes with this test already built for CUDA 11\.0\. You can similarly run your own script with EFA\. 

When constructing your own script, refer to the following guidance:
+ Provide the FI\_PROVIDER="efa" flag to enable EFA use\.
+ Use the complete path to mpirun as shown in the example while running NCCL applications with EFA\.
+ Change the params np and N based on the number of instances and GPUs in your cluster\.
+ Add the NCCL\_DEBUG=INFO flag and make sure that the logs indicate EFA usage as "Selected Provider is EFA"\.
+ Add the `FI_EFA_USE_DEVICE_RDMA=1` flag to use EFA's RDMA functionality for one\-sided and two\-sided transfer\.

Use the command `watch nvidia-smi` on any of the member nodes to monitor GPU usage\. The following `watch nvidia-smi` commands are for CUDA 11\.0 and depend on the Operating System of your instance\.
+ Amazon Linux 2:

  ```
  $ /opt/amazon/openmpi/bin/mpirun \
           -x FI_PROVIDER="efa" -n 16 -N 8 \
           -x NCCL_DEBUG=INFO \
           -x FI_EFA_USE_DEVICE_RDMA=1 -x NCCL_ALGO=ring -x --mca pml ^cm \
           -x LD_LIBRARY_PATH=/usr/local/cuda-11.0/efa/lib:/usr/local/cuda-11.0/lib:/usr/local/cuda-11.0/lib64:/usr/local/cuda-11.0:/opt/amazon/efa/lib64:/opt/amazon/openmpi/lib64:$LD_LIBRARY_PATH \
           --hostfile hosts --mca btl tcp,self --mca btl_tcp_if_exclude lo,docker0 --bind-to none \
           $HOME/src/bin/efa-tests/efa-cuda-11.0/all_reduce_perf -b 8 -e 1G -f 2 -g 1 -c 1 -n 100
  ```
+ Ubuntu 16\.04 and Ubuntu 18\.04:

  ```
  $ /opt/amazon/openmpi/bin/mpirun \
           -x FI_PROVIDER="efa" -n 16 -N 8 \
           -x NCCL_DEBUG=INFO \
           -x FI_EFA_USE_DEVICE_RDMA=1 -x NCCL_ALGO=ring -x --mca pml ^cm \
           -x LD_LIBRARY_PATH=/usr/local/cuda-11.0/efa/lib:/usr/local/cuda-11.0/lib:/usr/local/cuda-11.0/lib64:/usr/local/cuda-11.0:/opt/amazon/efa/lib:/opt/amazon/openmpi/lib:$LD_LIBRARY_PATH \
           --hostfile hosts --mca btl tcp,self --mca btl_tcp_if_exclude lo,docker0 --bind-to none \
           $HOME/src/bin/efa-tests/efa-cuda-11.0/all_reduce_perf -b 8 -e 1G -f 2 -g 1 -c 1 -n 100
  ```

Your output should look like the following\.

```
# nThread 1 nGpus 1 minBytes 8 maxBytes 1073741824 step: 2(factor) warmup iters: 5 iters: 100 validation: 1 
    #
    # Using devices
    #   Rank  0 Pid  18546 on ip-172-31-70-88 device  0 [0x10] A100-SXM4-40GB
    #   Rank  1 Pid  18547 on ip-172-31-70-88 device  1 [0x10] A100-SXM4-40GB
    #   Rank  2 Pid  18548 on ip-172-31-70-88 device  2 [0x20] A100-SXM4-40GB
    #   Rank  3 Pid  18549 on ip-172-31-70-88 device  3 [0x20] A100-SXM4-40GB
    #   Rank  4 Pid  18550 on ip-172-31-70-88 device  4 [0x90] A100-SXM4-40GB
    #   Rank  5 Pid  18551 on ip-172-31-70-88 device  5 [0x90] A100-SXM4-40GB
    #   Rank  6 Pid  18552 on ip-172-31-70-88 device  6 [0xa0] A100-SXM4-40GB
    #   Rank  7 Pid  18556 on ip-172-31-70-88 device  7 [0xa0] A100-SXM4-40GB
    #   Rank  8 Pid  19502 on ip-172-31-78-249 device  0 [0x10] A100-SXM4-40GB
    #   Rank  9 Pid  19503 on ip-172-31-78-249 device  1 [0x10] A100-SXM4-40GB
    #   Rank 10 Pid  19504 on ip-172-31-78-249 device  2 [0x20] A100-SXM4-40GB
    #   Rank 11 Pid  19505 on ip-172-31-78-249 device  3 [0x20] A100-SXM4-40GB
    #   Rank 12 Pid  19506 on ip-172-31-78-249 device  4 [0x90] A100-SXM4-40GB
    #   Rank 13 Pid  19507 on ip-172-31-78-249 device  5 [0x90] A100-SXM4-40GB
    #   Rank 14 Pid  19508 on ip-172-31-78-249 device  6 [0xa0] A100-SXM4-40GB
    #   Rank 15 Pid  19509 on ip-172-31-78-249 device  7 [0xa0] A100-SXM4-40GB
    ip-172-31-70-88:18546:18546 [0] NCCL INFO Bootstrap : Using [0]eth0:172.31.71.137<0> [1]eth1:172.31.70.88<0> [2]eth2:172.31.78.243<0> [3]eth3:172.31.77.226<0>
    ip-172-31-70-88:18546:18546 [0] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
    ip-172-31-70-88:18546:18546 [0] NCCL INFO NET/OFI Selected Provider is efa
    ip-172-31-70-88:18546:18546 [0] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v3 symbol.
    ip-172-31-70-88:18546:18546 [0] NCCL INFO Using network AWS Libfabric
    NCCL version 2.7.8+cuda11.0
    ip-172-31-70-88:18552:18552 [6] NCCL INFO Bootstrap : Using [0]eth0:172.31.71.137<0> [1]eth1:172.31.70.88<0> [2]eth2:172.31.78.243<0> [3]eth3:172.31.77.226<0>
    ip-172-31-70-88:18552:18552 [6] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
    ip-172-31-70-88:18551:18551 [5] NCCL INFO Bootstrap : Using [0]eth0:172.31.71.137<0> [1]eth1:172.31.70.88<0> [2]eth2:172.31.78.243<0> [3]eth3:172.31.77.226<0>
    ip-172-31-70-88:18556:18556 [7] NCCL INFO Bootstrap : Using [0]eth0:172.31.71.137<0> [1]eth1:172.31.70.88<0> [2]eth2:172.31.78.243<0> [3]eth3:172.31.77.226<0>
    ip-172-31-70-88:18548:18548 [2] NCCL INFO Bootstrap : Using [0]eth0:172.31.71.137<0> [1]eth1:172.31.70.88<0> [2]eth2:172.31.78.243<0> [3]eth3:172.31.77.226<0>
    ip-172-31-70-88:18550:18550 [4] NCCL INFO Bootstrap : Using [0]eth0:172.31.71.137<0> [1]eth1:172.31.70.88<0> [2]eth2:172.31.78.243<0> [3]eth3:172.31.77.226<0>
    ip-172-31-70-88:18552:18552 [6] NCCL INFO NET/OFI Selected Provider is efa
    ip-172-31-70-88:18552:18552 [6] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v3 symbol.
    ip-172-31-70-88:18552:18552 [6] NCCL INFO Using network AWS Libfabric
    ip-172-31-70-88:18556:18556 [7] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
    ip-172-31-70-88:18548:18548 [2] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
    ip-172-31-70-88:18550:18550 [4] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
    ip-172-31-70-88:18551:18551 [5] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
    ip-172-31-70-88:18547:18547 [1] NCCL INFO Bootstrap : Using [0]eth0:172.31.71.137<0> [1]eth1:172.31.70.88<0> [2]eth2:172.31.78.243<0> [3]eth3:172.31.77.226<0>
    ip-172-31-70-88:18549:18549 [3] NCCL INFO Bootstrap : Using [0]eth0:172.31.71.137<0> [1]eth1:172.31.70.88<0> [2]eth2:172.31.78.243<0> [3]eth3:172.31.77.226<0>
    ip-172-31-70-88:18547:18547 [1] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
    ip-172-31-70-88:18549:18549 [3] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /usr/local/cuda-11.0/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
    ip-172-31-70-88:18547:18547 [1] NCCL INFO NET/OFI Selected Provider is efa
    ip-172-31-70-88:18547:18547 [1] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v3 symbol.
    ip-172-31-70-88:18547:18547 [1] NCCL INFO Using network AWS Libfabric
    ip-172-31-70-88:18549:18549 [3] NCCL INFO NET/OFI Selected Provider is efa
    ip-172-31-70-88:18549:18549 [3] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v3 symbol.
    ip-172-31-70-88:18549:18549 [3] NCCL INFO Using network AWS Libfabric
    ip-172-31-70-88:18551:18551 [5] NCCL INFO NET/OFI Selected Provider is efa
    ip-172-31-70-88:18551:18551 [5] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v3 symbol.
    ip-172-31-70-88:18551:18551 [5] NCCL INFO Using network AWS Libfabric
    ip-172-31-70-88:18556:18556 [7] NCCL INFO NET/OFI Selected Provider is efa
    ip-172-31-70-88:18556:18556 [7] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v3 symbol.
    ip-172-31-70-88:18556:18556 [7] NCCL INFO Using network AWS Libfabric
    ip-172-31-70-88:18548:18548 [2] NCCL INFO NET/OFI Selected Provider is efa
    ip-172-31-70-88:18548:18548 [2] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v3 symbol.
    ip-172-31-70-88:18548:18548 [2] NCCL INFO Using network AWS Libfabric
    ip-172-31-70-88:18550:18550 [4] NCCL INFO NET/OFI Selected Provider is efa
    ip-172-31-70-88:18550:18550 [4] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v3 symbol.
    ip-172-31-70-88:18550:18550 [4] NCCL INFO Using network AWS Libfabric
-----------------------------some output truncated-----------------------------------
    #
    #                                                     out-of-place                       in-place          
    #       size         count    type   redop     time   algbw   busbw  error     time   algbw   busbw  error
    #        (B)    (elements)                     (us)  (GB/s)  (GB/s)            (us)  (GB/s)  (GB/s)       
    ip-172-31-70-88:18546:18546 [0] NCCL INFO Launch mode Parallel
               8             2   float     sum    158.9    0.00    0.00  2e-07    158.1    0.00    0.00  1e-07
              16             4   float     sum    158.3    0.00    0.00  1e-07    159.3    0.00    0.00  1e-07
              32             8   float     sum    157.8    0.00    0.00  1e-07    158.1    0.00    0.00  1e-07
              64            16   float     sum    158.7    0.00    0.00  1e-07    158.4    0.00    0.00  6e-08
             128            32   float     sum    160.2    0.00    0.00  6e-08    158.8    0.00    0.00  6e-08
             256            64   float     sum    159.8    0.00    0.00  6e-08    159.8    0.00    0.00  6e-08
             512           128   float     sum    161.7    0.00    0.01  6e-08    161.7    0.00    0.01  6e-08
            1024           256   float     sum    177.8    0.01    0.01  5e-07    177.4    0.01    0.01  5e-07
            2048           512   float     sum    198.1    0.01    0.02  5e-07    198.1    0.01    0.02  5e-07
            4096          1024   float     sum    226.2    0.02    0.03  5e-07    225.8    0.02    0.03  5e-07
            8192          2048   float     sum    249.3    0.03    0.06  5e-07    249.4    0.03    0.06  5e-07
           16384          4096   float     sum    250.4    0.07    0.12  5e-07    251.0    0.07    0.12  5e-07
           32768          8192   float     sum    256.7    0.13    0.24  5e-07    257.2    0.13    0.24  5e-07
           65536         16384   float     sum    269.8    0.24    0.46  5e-07    271.2    0.24    0.45  5e-07
          131072         32768   float     sum    288.3    0.45    0.85  5e-07    286.8    0.46    0.86  5e-07
          262144         65536   float     sum    296.1    0.89    1.66  5e-07    295.6    0.89    1.66  5e-07
          524288        131072   float     sum    376.7    1.39    2.61  5e-07    382.0    1.37    2.57  5e-07
         1048576        262144   float     sum    448.6    2.34    4.38  5e-07    451.1    2.32    4.36  5e-07
         2097152        524288   float     sum    620.2    3.38    6.34  5e-07    615.9    3.41    6.38  5e-07
         4194304       1048576   float     sum    768.2    5.46   10.24  5e-07    759.8    5.52   10.35  5e-07
         8388608       2097152   float     sum   1228.5    6.83   12.80  5e-07   1223.3    6.86   12.86  5e-07
        16777216       4194304   float     sum   2002.7    8.38   15.71  5e-07   2004.5    8.37   15.69  5e-07
        33554432       8388608   float     sum   2988.8   11.23   21.05  5e-07   3012.0   11.14   20.89  5e-07
        67108864      16777216   float     sum   8072.1    8.31   15.59  5e-07   8102.4    8.28   15.53  5e-07
       134217728      33554432   float     sum    11431   11.74   22.01  5e-07    11474   11.70   21.93  5e-07
       268435456      67108864   float     sum    17603   15.25   28.59  5e-07    17641   15.22   28.53  5e-07
       536870912     134217728   float     sum    35110   15.29   28.67  5e-07    35102   15.29   28.68  5e-07
      1073741824     268435456   float     sum    70231   15.29   28.67  5e-07    70110   15.32   28.72  5e-07
    # Out of bounds values : 0 OK
    # Avg bus bandwidth    : 7.14456
```