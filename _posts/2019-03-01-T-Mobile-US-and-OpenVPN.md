---
layout: post
title: T-Mobile US and OpenVPN
---

Recently, I switched my familie's mobile phone plan to T-Mobile and this has caused some issues with the VPN connection to my home from my phone. At times, the VPN would not come up at all and other times the VPN would connect just fine, but no data would flow. Even stranger were the times that the VPN connection came up and appeared to work fine until I downloaded a file or attempted to play some media from my [Emby](https://emby.media) server.

After much testing and googling around, I finally found that the issue had to do with the MTU (essentially: maximum packet size) via T-Mobile. I'm not quite sure of the specific details, but as far as I've been able to determine the problem is that T-Mobile uses a 4-over-6 tunneling system to tunnel IPv4 traffic over their IPv6-only mobile network. Whether that is true or not can only be determined by T-Mobile. They are not very helpful on their forums, referring to the fact that VPN is unsupported and they cannot give any technical details or assistance. Packets that are larger than some threshold appear to get blocked or lost in the T-Mobile network and cause the VPN connection to stall, breaking file transfers and media playback.

Lowering the maximum packet size via [OpenVPN configuration](https://openvpn.net/community-resources/reference-manual-for-openvpn-2-4/) turned out to be the solution to my problem. Simply add the following parameter to your client's .ovpn file to lower the MTU for the connection:

```
mssfix 1400
```

The default value is 1450, which resulted in packets that were too large. Setting this to 1400 solved my problem. You may need a lower value or be able to use a higher one depending on various factors, your mileage may vary and some experimentation may be required to determine the correct value for your situation.
