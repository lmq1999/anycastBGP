# Anycast BGP lab 

## Mô hình

![model](https://raw.githubusercontent.com/lmq1999/anycastBGP/main/image/architechture.png)

## Cấu hình

### Cấu hình server

1. Đặt IP anycast ở loopback interface

    ```bash
    lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
        inet 10.10.10.10/32 scope global lo
        valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
        valid_lft forever preferred_lft forever
    ```

2. Sử dụng quagga và zebra để cấu hình bgp để pair với L3 switch

    **Server 1 /etc/quagga/bgpd.conf**

    ```
    !
    ! Zebra configuration saved from vty
    !   2021/09/27 08:12:16
    !
    !
    router bgp 65001                                # AS của server
    bgp router-id 192.168.1.5                       # ID server
    network 10.10.10.0/24                           # network quảng bá qua bgp (ip anycast)
    neighbor 192.168.1.2 remote-as 65000            # neighbor 
    neighbor 192.168.1.2 interface ens3
    !
    address-family ipv6
    exit-address-family
    exit
    !
    route-map RM_SET_SRC permit 10
    !
    line vty
    !
    ```

    **Server 1/etc/quagga/zebra.conf**
    
    ```
    !
    ! Zebra configuration saved from vty
    !   2021/09/27 08:12:16
    !
    !
    interface ens3
    !
    interface ens4
    !
    interface lo
    !
    route-map RM_SET_SRC permit 10
    set src 192.168.1.5
    !
    router-id 192.168.1.5
    !
    ip protocol bgp route-map RM_SET_SRC
    !
    line vty
    !
    ```


    **Server 2 /etc/quagga/bgpd.conf**

    ```
    !
    ! Zebra configuration saved from vty
    !   2021/09/27 08:12:16
    !
    !
    router bgp 65001                                # AS của server
    bgp router-id 192.168.1.6                       # ID server
    network 10.10.10.0/24                           # network quảng bá qua bgp (ip anycast)
    neighbor 192.168.1.2 remote-as 65000            # neighbor 
    neighbor 192.168.1.2 interface ens3
    !
    address-family ipv6
    exit-address-family
    exit
    !
    route-map RM_SET_SRC permit 10
    !
    line vty
    !
    ```

    **Server 2 /etc/quagga/zebra.conf**
    
    ```
    !
    ! Zebra configuration saved from vty
    !   2021/09/27 08:12:16
    !
    !
    interface ens3
    !
    interface ens4
    !
    interface lo
    !
    route-map RM_SET_SRC permit 10
    set src 192.168.1.6
    !
    router-id 192.168.1.6
    !
    ip protocol bgp route-map RM_SET_SRC
    !
    line vty
    !
    ```

### Cấu hình L3 Switch

Cấu hình các interface như bình thường.

Cấu hình bgp để pair với 2 server, đồng thời để multipath để LB.
```
router bgp 65000
bgp log-neighbor-changes
neighbor 192.168.1.5 remote-as 65001
neighbor 192.168.1.6 remote-as 65001
!        
address-family ipv4
network 192.168.1.2
neighbor 192.168.1.5 activate
neighbor 192.168.1.6 activate
maximum-paths 4
auto-summary
exit-address-family
```

## Kết quả 

### Ping

Khi client ping đến IP anycast, packet được chia đều cho 2 backend 

![ping1](https://raw.githubusercontent.com/lmq1999/anycastBGP/main/image/ping_result_1.png)

![ping2](https://raw.githubusercontent.com/lmq1999/anycastBGP/main/image/ping_result_2.png)

### DNS querry

Khi client sử dụng DNS từ 2 server với anycast IP

request ở trên cả 2 server

![dns1](https://raw.githubusercontent.com/lmq1999/anycastBGP/main/image/dns_result_1.png)

![dns2](https://raw.githubusercontent.com/lmq1999/anycastBGP/main/image/dns_result_2.png)
