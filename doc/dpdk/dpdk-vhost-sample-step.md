1. Install Dependency
Kernel Version(Linux 6.12.19-1-lts x86_64)
```bash
sudo apt install -y meson ninja-build python3-pyelftools libnuma-dev pkg-config
git clone https://github.com/DPDK/dpdk.git
cd dpdk
git checkout v24.11
meson setup -Dexamples=all build
ninja -C build install
ldconfig
```
2. Setup HugePage
```bash
./dpdk-hugpages.py --setup 8G
```
3. Bind NIC
```bash
./dpdk-devbind.py -b vfio-pci {PCI addr}
```
4. Start DPDK vhost
```bash
./dpdk-vhost -l 0-3 \
    --vdev 'net_vhost0,iface=/tmp/vm1.sock,vlan=100' \
    --vdev 'net_vhost1,iface=/tmp/vm2.sock,vlan=200' \
    -- --stats 1
```
Common problem:
    1. portmask mismatch, use -- -p to adjust
    1. IOMMU group issue
        1. adjust IOMMU group by moving hardware into the correct PCI slot
        1. For demo just use no-iommu-mode. Reading: linux_drivers.html#vfio-no-iommu-mode (Outdated No longer working in newer kernel)
5. Start Qemu
```bash
qemu-system-x86_64 \
    -cpu host \
    -enable-kvm \
    -m 4G \
    -netdev type=vhost-user,id=net0,chardev=char0 \
    -chardev socket,id=char0,path=/tmp/vm1.sock,server=on \
    -device virtio-net-pci,netdev=net0,mac={virtio mac} \

qemu-system-x86_64 \
    -cpu host \
    -enable-kvm \
    -m 4G \
    -netdev type=vhost-user,id=net0,chardev=char0 \
    -chardev socket,id=char0,path=/tmp/vm1.sock,server=on \
    -device virtio-net-pci,netdev=net0,mac={virtio mac} \
```
6. Test
    1. VM handle vlan tag
    ```bash
    # VM1
    ip link add link eth0 name eth0.100 type vlan id 100
    ip addr add 192.168.100.2/24 dev eth0.100
    # VM2
    ip link add link eth0 name eth0.200 type vlan id 200
    ip addr add 192.168.200.2/24 dev eth0.200
    ```
    1. use dpdk-testpmd to send packet to host
7. Change vhost to use HW NIC(Need verification)
```C
struct rte_eth_conf port_conf = {
    .rxmode = {
        .mq_mode = ETH_MQ_RX_NONE,
        .max_rx_pkt_len = RTE_ETHER_MAX_LEN,
    },
    .txmode = {
        .mq_mode = ETH_MQ_TX_NONE,
    },
};

ret = rte_eth_dev_configure(0, 1, 1, &port_conf);
if (ret < 0)
    rte_exit(EXIT_FAILURE, "Failed to configure Huawei HiNIC port\n");

ret = rte_eth_rx_queue_setup(0, 0, 128, rte_eth_dev_socket_id(0), NULL, mbuf_pool);
ret = rte_eth_tx_queue_setup(0, 0, 512, rte_eth_dev_socket_id(0), NULL);

rte_eth_dev_start(0);
```
compile then repeat from step 4.
