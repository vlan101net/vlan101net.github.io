---
layout: post
title:  "FortiGate ADVPN 2.0"
date:   2025-11-20 09:07:48 -0600
categories: jekyll update
---



## Hub Configuration
{% highlight fortios %}
config system settings
    set location-id 198.51.100.4
    set tcp-session-without-syn enable
    set allow-subnet-overlap enable
end

config system interface
    edit "Loopback0"
        set vdom "root"
        set ip 198.51.100.4/32
        set allowaccess ping
        set type loopback
        set role lan
    next
end

config vpn ipsec phase1-interface
    edit "HUB-WAN1"
        set type dynamic
        set interface "wan1"
        set ike-version 2
        set net-device disable
        set dpd on-idle
        set auto-discovery-sender enable
        set network-overlay enable
        set network-id 1
        set proposal aes256-sha256
        set dhgrp 14
        set psksecret key123!
        set dpd-retryinterval 5
        set exchange-ip-addr4 198.51.100.4
        set add-route disable
    next
    edit "HUB-WAN2"
        set type dynamic
        set interface "wan2"
        set ike-version 2
        set net-device disable
        set dpd on-idle
        set auto-discovery-sender enable
        set network-overlay enable
        set network-id 2
        set proposal aes256-sha256
        set dhgrp 14
        set psksecret key123!
        set dpd-retryinterval 5
        set exchange-ip-addr4 198.51.100.4
        set add-route disable
    next
end

config vpn ipsec phase2-interface
    edit "HUB-WAN1-P2"
        set phase1name "HUB-WAN1"
        set proposal aes256-sha256
    next
    edit "HUB-WAN2-P2"
        set phase1name "HUB-WAN2"
        set proposal aes256-sha256
    next
end

config system sdwan
    config zone
        edit "BRANCHES"
        next
        edit "INET.Access"
        next
    end
    config members
        edit 1
            set interface port1
            set gateway 1.1.1.1
            set zone INET.Access
        next
        edit 2
            set interface port2
            set gateway 8.8.8.8
            set zone INET.Access
        next
        edit 3
            set interface HUB-WAN1
            set zone BRANCHES
        next
        edit 4
            set interface HUB-WAN2
            set zone BRANCHES
        next
    end
end

config firewall policy
    edit 0
        set name "Loopback0.to.ADVPN"
        set srcintf Loopback0
        set dstintf BRANCHES
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
    next
    edit 0
        set name "ADVPN.to.Loopback0"
        set dstintf Loopback0
        set srcintf BRANCHES
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
    next
end

config router bgp
    set as 65400
    set router-id 198.51.100.4
    set ibgp-multipath enable
    set recursive-next-hop enable
    set tag-resolve-mode merge
    config neighbor-group
        edit "ADVPN"
            set soft-reconfiguration enable
            set capability-graceful-restart enable
            set advertisement-interval 1
            set connect-timer 1
            set interface "Loopback0"
            set remote-as 65400
            set update-source "Loopback0"
            set route-reflector-client enable
        next
    end
    config neighbor-range
        edit 1
            set prefix 198.51.100.0 255.255.255.0
            set neighbor-group "ADVPN"
        next
    end
    config network
        edit 1
            set prefix 198.51.100.0 255.255.255.0
        next
    end
end

config router static
    edit 0           
        set dst 198.51.100.0 255.255.255.0
        set distance 252
        set blackhole enable
    next
end
{% endhighlight %}

## Spoke Configuration
{% highlight fortios %}
config system settings
  set location-id 198.51.100.2
  set tcp-session-without-syn enable
  set allow-subnet-overlap enable
end

config system interface
    edit "Loopback0"
        set vdom "root"
        set ip 198.51.100.2/32
        set allowaccess ping
        set type loopback
        set role lan
    next
end

config vpn ipsec phase1-interface
    edit "SPOKE-WAN1"
        set interface "wan1"
        set ike-version 2
        set net-device enable
        set proposal aes256-sha256
        set dpd on-idle
        set dhgrp 14
        set idle-timeout enable
        set idle-timeoutinterval 5
        set auto-discovery-receiver enable
        set network-overlay enable
        set network-id 1
        set remote-gw 1.1.1.2
        set psksecret key123!
        set dpd-retryinterval 5
        set exchange-ip-addr4 198.51.100.2
        set add-route disable
    next
    edit "SPOKE-WAN2"
        set interface "wan2"
        set ike-version 2
        set net-device enable
        set proposal aes256-sha256
        set dpd on-idle
        set dhgrp 14
        set idle-timeout enable
        set idle-timeoutinterval 5
        set auto-discovery-receiver enable
        set network-overlay enable
        set network-id 2
        set remote-gw 8.8.8.8.
        set psksecret key123!
        set dpd-retryinterval 5
        set exchange-ip-addr4 198.51.100.2
        set add-route disable
    next
end

config vpn ipsec phase2-interface
    edit "SPOKE-WAN1-P2"
        set phase1name "SPOKE-WAN1"
        set proposal aes256-sha256
    next
    edit "SPOKE-WAN2-P2"
        set phase1name "SPOKE-WAN2"
        set proposal aes256-sha256
    next
end

config router prefix-list
    edit LOOPBACK-ADDR
        config rule
            edit 0
                set action permit
                set prefix 198.51.100.2/32
                unset ge
                unset le
            next
        end
    next
    edit ATL-NETS
        config rule
            edit 0
                set action permit
                set prefix 10.2.10.0/24
                unset le
                unset ge
            next
            edit 0
                set action permit
                set prefix 10.93.6.0/24
                unset le
                unset ge
            next
            edit 0
                set action permit
                set prefix 10.93.60.0/24
                unset le
                unset ge
            next
            edit 0
                set action permit
                set prefix 10.93.61.0/24
                unset le
                unset ge
            next
            edit 0
                set action permit
                set prefix 10.6.0.0/16
                set le 32
                unset ge
            next
            edit 0
                set action permit
                set prefix 10.2.0.0/16
                set le 32
                unset ge
            next
        end
    next
end

config router route-map
    edit "SET-TAG-1"
        config rule
            edit 1
                set set-tag 1
            next
        end
    next
    edit "ADVPN-OUT"
        config rule
            edit 1
                set match-ip-address "ATL-NETS"
            next
            edit 2
                set match-ip-address "LOOPBACK-ADDR"
            next
        end
    next
end

config router bgp
    set as 65400
    set router-id 198.51.100.2
    set ibgp-multipath enable
    set recursive-next-hop enable
    set tag-resolve-mode merge
    config neighbor
        edit "198.51.100.4"
            set soft-reconfiguration enable
            set capability-graceful-restart enable
            set advertisement-interval 1
            set connect-timer 1
            set interface "Loopback0"
            set remote-as 65400
            set route-map-in "SET-TAG-1"
            set route-map-out "ADVPN-OUT"
            set update-source "Loopback0"
            set additional-path both
        next
    end
    config network
        edit 2
            set prefix 198.51.100.2 255.255.255.255
        next
    end
    config redistribute connected
        set status enable
    end
    config redistribute static
        set status enable
    end
end

config system sdwan
    config zone
        edit "BRANCHES"
        next
        edit "INET.Access"
        next
    end
    config members
        edit 1
            set interface wan1
            set gateway 4.4.4.4
            set zone INET.Access
        next
        edit 2
            set interface wan2
            set zone INET.Access
        next
        edit 3
            set interface SPOKE-WAN1
            set zone BRANCHES
        next
        edit 4
            set interface SPOKE-WAN2
            set zone BRANCHES
        next
    end
    config health-check
        edit "HUB.MONITOR"
            set server 10.4.11.1
            set embed-measured-health enable
            set source 198.51.100.2
            set members 3 4
            config sla
                edit 0
                    set link-cost-factor latency packet-loss
                    set latency-threshold 250
                    set packetloss-threshold 5
                next
            end
        next
    end
end

config firewall policy
    edit 0
        set name "Loopback0.to.ADVPN"
        set srcintf Loopback0
        set dstintf ADVPN
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
    next
    edit 0
        set name "ADVPN.to.Loopback0"
        set dstintf Loopback0
        set srcintf ADVPN
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
    next
end
{% endhighlight %}