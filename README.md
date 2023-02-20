# sqm-scripts-nss
Smart Queue Management Scripts for OpenWRT for use with NSS optimized builds.

NSS FQ-Codel proves very effective at maintaining low latency under load, while causing minimal CPU load on the router. 

Currently only supports nssfq-codel and no traffic classification / marking due to limitations of the current driver. 

## Requirements

* An SQM enabled OpenWRT build such as https://github.com/ACwifidude/openwrt
* sqm-scripts package
* luci-app-sqm package (for configuration from the GUI)

## Installation

### Manual Installation

* Just copy the nss-rk.qos and nss-rk.qos.help files to /usr/lib/sqm on your router

### Package Installation

* Download the .ipk package file from the [releases page](https://github.com/rickkdotnet/sqm-scripts-nss/releases/tag/ipk)
* Go to the System -> Software menu on your router and upload the .ipk package file

### Installation via feeds

If you're building OpenWRT yourself, you can add this script to your build with a feed: 

    echo "src-git sqm_scripts_nss https://github.com/rickkdotnet/sqm-scripts-nss.git >> feeds.conf
    ./scripts/feeds update
    ./scripts/feeds install sqm-scripts-nss
 
 Now you can find the script in menuconfig under 'Extra packages'.

## Configuration 

* Go to Network -> SQM QoS in luci
* Add a a queue or change your existing one
* Select your physical uplink interface (usually eth0)  
* Sheck the 'enable this SQM instance' checkbox
* Enter your down and upload speeds, 95% of your actual line speed is a good ballpark figure
* Go to the queue discipline tab and select "fq_codel" as discipline and and "nss-rk.qos" for the script
* Configure other parameters if you want, although the defaults should work well. If you like to play with the codel interval, you can do by entering 'interval XXms' in the 'advanced option string'. 
* Click "save and apply" 

If all went well you should now be able to enjoy lower latency under load, with minimal CPU load on the router. 

If it's working the output from tc should look something like this: 

    root@nighthawk:~# tc -s qdisc show dev eth0
    qdisc nsstbl 1: root refcnt 2 buffer/maxburst 4500b rate 36Mbit mtu 1514b accel_mode 0
     Sent 2692502 bytes 7109 pkt (dropped 0, overlimits 190 requeues 0)
     backlog 0b 0p requeues 0
    qdisc nssfq_codel 10: parent 1: target 5ms limit 1001p interval 50ms flows 1024 quantum 300 set_default accel_mode 0
     Sent 2692502 bytes 7109 pkt (dropped 0, overlimits 0 requeues 0)
     backlog 0b 0p requeues 0
     maxpacket 1518 drop_overlimit 0 new_flow_count 4024 ecn_mark 0
     new_flows_len 0 old_flows_len 5

    root@nighthawk:~# tc -s qdisc show dev nssifb
    qdisc nsstbl 1: root refcnt 2 buffer/maxburst 45000b rate 360Mbit mtu 1514b accel_mode 0
     Sent 2182202 bytes 8391 pkt (dropped 0, overlimits 0 requeues 0)
     backlog 0b 0p requeues 0
    qdisc nssfq_codel 10: parent 1: target 5ms limit 1001p interval 50ms flows 1024 quantum 1514 set_default accel_mode 0
     Sent 2182202 bytes 8391 pkt (dropped 0, overlimits 0 requeues 0)
     backlog 0b 0p requeues 0
     maxpacket 1518 drop_overlimit 0 new_flow_count 5081 ecn_mark 0
     new_flows_len 0 old_flows_len 1



## Known bugs, limitations


* Due to limitations of the driver:
    * Only fq-codel is supported as a queue discipline
    * No marking or traffic classification is currently possible, this also means that DSCP squashing does not work
    * ECN marking is not supported
* The script does not does anything with the Link Layer Adaptation fields. 
* On kernel 5.10 the script does not remove the nssifb interface if it's stopped, because removing or even bringing down the interface frequently crashed my router. This could cause problems if you're switching to a script which set up regular ifb4ethX interfaces. You probably need to reboot if you want to switch to another SQM script. On kernel 5.15 this problem has been resolved. 

