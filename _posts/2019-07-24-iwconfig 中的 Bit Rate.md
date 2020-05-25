---
title: iwconfig 中的 Bit Rate
date: 2019-07-24
tags: [linux,network,tools]
categories: [linux,network,tools]
---

最近的业务都涉及到局域网通信，但是业务所处的无线网络非常不稳定，因此需要对当前网络情况进行一个探测，其中一个网络性能指标就是当前网络的传输带宽（指局域网中多个设备之间的传输带宽），`iperf`是一个较好的带宽测试工具，它通过在网络中发送数据包来测量网络带宽，但是最近发现`iwconfig`中也有一个指标叫做`Bit Rate`,感觉就是我要的网络带宽，而且它还会根据每次测量变化，看起来真的很像我们所需要的数据。以下研究证明，事情并没有想象的那么简单。

## 问题描述

`iwconfig` 类似 `ifconfig`，只不过它专门用来查看无线网络，`iwconfig` 中的 w 就表示 wireless。

直接执行 iwconfig 的结果如下(执行了3遍)：

```sh
$ iwconfig
wlan0     IEEE 802.11abgn  ESSID:"NETGEAR29-5G-2"  
          Mode:Managed  Frequency:5.765 GHz  Access Point: 14:59:C0:93:94:17   
          Bit Rate=433.3 Mb/s   Tx-Power=22 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Encryption key:off
          Power Management:on
          Link Quality=51/70  Signal level=-59 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:188   Missed beacon:0
$ iwconfig
wlan0     IEEE 802.11abgn  ESSID:"NETGEAR29-5G-2"  
          Mode:Managed  Frequency:5.765 GHz  Access Point: 14:59:C0:93:94:17   
          Bit Rate=325 Mb/s   Tx-Power=22 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Encryption key:off
          Power Management:on
          Link Quality=54/70  Signal level=-56 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:187   Missed beacon:0
$ iwconfig
wlan0     IEEE 802.11abgn  ESSID:"NETGEAR29-5G-2"  
          Mode:Managed  Frequency:5.765 GHz  Access Point: 14:59:C0:93:94:17   
          Bit Rate=390 Mb/s   Tx-Power=22 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Encryption key:off
          Power Management:on
          Link Quality=52/70  Signal level=-58 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:185   Missed beacon:0
```

可以看到其中的`Bit Rate`值都是不一样的，分别是433.3Mb/s,325Mb/s,390Mb/s，那这个值是否就是当前网卡的传输速度呢？否则它怎么会变化呢。

## 解决过程

于是网上一通查找，发现有提到这个值是网络的最大带宽，但如果是最大带宽怎么会变化呢？因此问题还没有解决。

于是`apt source`下载下来`iwconfig`的源码，想看一下这个值是怎么来的，在`iwconfig.c:79`文件中找到了获取方法，如下：

```c
// file: wireless.10.h
158 #define SIOCGIWRATE 0x8B21      /* get default bit rate (bps) */
    
// file: iwconfig.c
78   /* Get bit rate */
79   if(iw_get_ext(skfd, ifname, SIOCGIWRATE, &wrq) >= 0)                             80     {
81       info->has_bitrate = 1;
82       memcpy(&(info->bitrate), &(wrq.u.bitrate), sizeof(iwparam));
83     }

// file: iwlib.h
433 /*------------------------------------------------------------------*/
434 /* 
435  * Wrapper to extract some Wireless Parameter out of the driver
436  */
437 static inline __attribute__((always_inline)) int
438 iw_get_ext(int          skfd,       /* Socket to the kernel */
439        const char *     ifname,     /* Device name */
440        int          request,    /* WE ID */
441        struct iwreq *   pwrq)       /* Fixed part of the request */
442 {      
443   /* Set device name */
444   strncpy(pwrq->ifr_name, ifname, IFNAMSIZ);
445   /* Do the request */
446   return(ioctl(skfd, request, pwrq));
447 }
```

可以看到，`iw_get_ext`的`SIOCGIWRATE`参数用来获取Bit Rate，然后调用`ioctl`来获取值。

这下有些麻烦了，`ioctl`是用来和驱动打交道的，常用来和设备驱动打交道，要想知道它的数据来源需要查找对应驱动的实现，先不管了，由于这个数据几乎所有的无线网卡都支持，因此应该随便找一个驱动就行了，接下来就要去内核中找一个驱动随便来看一下，由于我使用RTL系列的网卡，因此我在内核中找到了rtl8188eu的网卡驱动，经过查找，找到了它的实现如下：

```c
/*
* rtw_get_cur_max_rate -
* @adapter: pointer to struct adapter structure
*
* Return 0 or 100Kbps
*/
u16 rtw_get_cur_max_rate(struct adapter *adapter)
{
...
            /* cur_bwmod is updated by beacon, pmlmeinfo is updated by association response */
            bw_40MHz = (pmlmeext->cur_bwmode && (HT_INFO_HT_PARAM_REC_TRANS_CHNL_WIDTH & pmlmeinfo->HT_info.infos[0])) ? 1 : 0;

            short_GI_20 = (le16_to_cpu(pmlmeinfo->HT_caps.cap_info) & IEEE80211_HT_CAP_SGI_20) ? 1 : 0;
            short_GI_40 = (le16_to_cpu(pmlmeinfo->HT_caps.cap_info) & IEEE80211_HT_CAP_SGI_40) ? 1 : 0;

            max_rate = rtw_mcs_rate(                                                                                                  
                RF_1T1R,
                bw_40MHz & (pregistrypriv->cbw40_enable),
                short_GI_20,
                short_GI_40,
                pmlmeinfo->HT_caps.mcs.rx_mask
            );  
        }   
...
    return max_rate;
}

/* show MCS rate, unit: 100Kbps */
u16 rtw_mcs_rate(u8 rf_type, u8 bw_40MHz, u8 short_GI_20, u8 short_GI_40, unsigned char *MCS_rate)                                    
{
    ...
        if (MCS_rate[0] & BIT(7))
            max_rate = (bw_40MHz) ? ((short_GI_40) ? 1500 : 1350) : ((short_GI_20) ? 722 : 650);
        else if (MCS_rate[0] & BIT(6)) 
            max_rate = (bw_40MHz) ? ((short_GI_40) ? 1350 : 1215) : ((short_GI_20) ? 650 : 585);
        else if (MCS_rate[0] & BIT(5)) 
            max_rate = (bw_40MHz) ? ((short_GI_40) ? 1200 : 1080) : ((short_GI_20) ? 578 : 520);
... 
    return max_rate;
}
```

这里面都是些数值计算，并没有任何发送数据包，接收数据包然后计算带宽的操作，因此，`iwconfig`

的结果很可能是通过计算来的而不是测量来的，顺着这个思路去查了一下802.11ac的Bit Rate，然后就查到了一张[速率对应表](https://community.cisco.com/t5/wireless-mobility-documents/802-11ac-mcs-rates/ta-p/3155920)，如下面这样：

![1564025404705](/data/1564025404705.png)

从这张表上可以看到我之前测得的325Mb/s,390Mb/s,433.3Mb/s都在这个表上，又多测了几次，果然速度都逃不过这张表，后来又用`iw wlan0 link`查了一下网卡信息，结果如下：

```sh
$ iw wlan0 link
...
	tx bitrate: 390.0 MBit/s VHT-MCS 8 80MHz short GI VHT-NSS 1

$ iw wlan0 link
...
	tx bitrate: 325.0 MBit/s VHT-MCS 7 80MHz short GI VHT-NSS 1

$ iw wlan0 link
...
	tx bitrate: 433.3 MBit/s VHT-MCS 9 80MHz short GI VHT-NSS 1

```

`iw`类似`ip`，和 `iwconfig` 的功能类似（要更强大一点），它结果显示出了一些更详细的参数 `VHT-MCS 8 80MHz short GI VHT-NSS 1`,这些参数就是表格中的表头了，也就是查表的参数了，`NSS 1`表示`Number of Special Stream`,也就是表的第一列中的数字1，`VHT-MCS 8 80MHz`就表示第7行的80MHz,`short GI`表示 `Short Guard interval(400ns) `,也就是`400ns GI`那一列，所以就在表中查到了 390，这个数字就是iw的返回结果了。其他数字也可以用相同的方式查表得到。

## 总结

iw后者iwconfig显示出来的速度是当前经过自动适配之后的网卡最大调制和编码速度，也就是MCS,这个值是通过`Link adaptation`适配出来的，并不是当前网卡的传输速度，只是表示当前网卡以这个速度调制和编码，当前也可以理解为当前的速度上限。



参考链接：

<https://unix.stackexchange.com/questions/76457/how-to-find-speed-of-wlan-interface>

<https://community.cisco.com/t5/wireless-mobility-documents/802-11ac-mcs-rates/ta-p/3155920>

<https://en.wikipedia.org/wiki/Link_adaptation>

<https://www.winncom.com/en/glossary/619/nss>

<https://www.wlanpros.com/mcs-index-charts/>

