T
RoCEv1
  - frame size 1500 or 9000 jumbo
  - lossless for performance
  - RDMA Traffic on Lossless priorities (PFC of ethernet) Paper: RoCE Rocks without PFC: Detailed Evaluation
  - Legacy Traffic on Lossy priorities

RoCEv2 \ routable Roce RRoCE
  - RoCE v2 protocol exists on top of either the UDP/IPv4 or the UDP/IPv6 protocol.[3] The UDP destination port number 4791 has been reserved for RoCE v2
  - L3 TOR per rack
  - Converged network: lossless + best-effort
  - PFC (Priority Flow Control)
    - vlan PCP
    - HOL, congestion spreading deadlocks
  - Congestion management
    - ECN/CNP IP
    - congestion control algorithm
      -  Data Center Quantized Congestion Notification (DCQCN)
    - CNP reactions and rate reduction/increase parameters
  - if no PFC and lossful network
      - loss recovery , end-to-end flow control
  - Improved RoCE NIC

- RoCE ethnet proformance
- IB transport protocol
  - Lossless
  - RoCE Congestion Management
  - Link Level Flow Control

Modes
  - Lossless
  - Semi-lossless
  - Lossy

iWARP?
IB retx all pacekts after OOO

Ethertype Based Traffic De-multiplexing (Converged NICs)
Stateless Traffic Identification (Switch/Fabric Monitoring, ACLs)


- performance degradation
  - Programming model and APIs, synchronization
  - Device and LAN, frame conflict
  - middle boxes (switch & router)
    - buffer overflow * congestion control
    - QoS configuration (IP DSCP VLAN PCP), PFC enabled?
    - Routing decision causes path unbalancing (ECMP)
  - OS scheduling & IO
    - Link utilization frame size and frequency
    - Fairness of bandwidth allocation between QPs

  - Hardware
      - CPU/GPU, Pipeline, memory bandith

from network issues to application performance using Zhang Cheng's example

Zhang Cheng proposed scenario vs tech issue and its monitoring

References

- [Nvidia RDMA Over Converged Ethernet (RoCE)](https://docs.nvidia.com/networking/pages/viewpage.action?pageId=39258700)
- [Nvida RoCE commands](https://docs.nvidia.com/networking/display/Onyxv391604/RoCE+Commands)
- [Nvida UNDERSTANDING ROCEV2 CONGESTION MANAGEMENT](https://enterprise-support.nvidia.com/s/article/understanding-rocev2-congestion-management)
- [Broadcom Validating RoCE Network on Linux](https://techdocs.broadcom.com/us/en/storage-and-ethernet-connectivity/ethernet-nic-controllers/bcm957xxx/adapters/Configuration-adapter/RoCE/validating-roce-network.html)
- [Broadcom RoCE statistics](https://techdocs.broadcom.com/us/en/storage-and-ethernet-connectivity/ethernet-nic-controllers/bcm957xxx/adapters/statistics/appendix-a-statistics-definitions.html#appendix-a-statistics-definitions)
- [Broadcom ethtool statistics](https://techdocs.broadcom.com/us/en/storage-and-ethernet-connectivity/ethernet-nic-controllers/bcm957xxx/adapters/statistics/roce-statistics.html)
- [Nvidia RoCE tools](https://enterprise-support.nvidia.com/s/article/roce-rdma-tools)
- [Data Center Quantized Congestion Notification (DCQCN)](https://www.juniper.net/documentation/us/en/software/junos/traffic-mgmt-qfx/topics/topic-map/cos-qfx-series-DCQCN.html)
- [Linux RDMA tool](https://man7.org/linux/man-pages/man8/rdma-statistic.8.html)
