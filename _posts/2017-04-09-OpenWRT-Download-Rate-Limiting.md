As a parent to an 11 year old girl with an iPhone (no mobile data contract), I have a problem. The problem is that, when given the chance, she will sit and watch Youtube all day - every day. There are a few ways to solve that problem:

1. Take the phone away
1. Closely monitor the phone use and take it away after x minutes/hours
1. Geek out

Being a geek, the only real option is of course the third one. Besides, option number 1 would result in a lot of drama and option number 2 is way too much work. My wife and I have better things to do than constantly monitor whether her x minutes/hours are already up and an 11 year old doesn't want nor should be under constant surveillance.

There are a few requirements to the solution that would work for us:

1. Watching videos needs to be possible but limited. We have no problem with watching videos but all day every day is not OK.
1. There should not be any need for manual action. If we have to take any manual actions there will be times that we forget and enforcement will be inconsistent, making it useless.
1. Messages through WhatsApp, iMessage, Instagram, etc should not be blocked. We live in suburbian USA and there is very limited mobility for teens here (no sidewalks, no bike roads so they have to be driven everywhere). Messages and such are a very important tool to stay in touch with friends socially.

Rate limiting seems to fit all these options. Her iPhone gets some quota (say 500MB) per day and after that the connection slows down to 16kB/s. This will not allow videos but message and email will work fine.

I looked around for ways of doing this with my OpenWRT router ([Turris Omnia](https://omnia.turris.cz/en/)) and ran into a few issues: Most of the rate limiting examples limit only upstream bandwidth and I want to mostly limit downstream instead. Some of the solutions used magic incantations for the SQM system (traffic shaping) that I found more complex than worth getting in to. I found the CONNMARK iptables target and it seems to fit the bill exactly:

First MARK all packets from my daughter's device:

```
iptables -t nat -A prerouting_rule -m mac --mac-source 1c:91:48:xx:xx:xx -j MARK --set-mark 0x0A -m comment --comment "iPhone SE"
```

If you want to include more devices in this rule, just add more MARK rules, you can add as many mac addresses as you like. When all desired packets are marked, we need to transfer this mark to the connection that they are a part of:

```
# If a packet is marked, make sure the connection is marked as well
iptables -t nat -A prerouting_rule -m mark --mark 0xA -j CONNMARK --save-mark

# On incoming packets, make sure to read the connection mark back into the packet
iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark
```

Because connection tracking is required for NAT anyway, this doesn't cause a lot of extra load on the router. All open connections from one of the marked devices will now be marked with the value 0xA, which we will use later to limit the download speed.

To do the actual rate limiting I used the hashlimit module. It can be found in opkg so it's easily installed on OpenWRT.

```
iptables  -A forwarding_rule -m mark --mark 0xA -m conntrack --ctstate ESTABLISHED,RELATED -m hashlimit --hashlimit-name "Over quota" --hashlimit-above 16kb/s -j DROP
```

*Please note: The above rules are all OpenWRT specific since the chain names used in them are created by the OpenWRT firewall scripts. Substitute appropriate chains if using this method on some other Linux distribution.*

Now that we have a rate limit in place, we need to be able to turn it on and off based on the amount of bandwidth consumed. There are ways of doing this with pure iptables but since those counters are not very reliable (they get wiped at every firewall reload), I decided to use [Majordomo](https://project.turris.cz/en/software) which was already included in my Turris Omnia router by default. It writes bandwidth statistics to /tmp in CSV format that we can easily use.

I very poorly cobbled together a [Python script](https://gist.github.com/gerco/e0d6d4876eb4d77e3dabfb3e6a3faf21) to check whether the quota has been exceeded and call it from cron every minute. This creates a marker file that I check for in /etc/firewall.user and creates the above iptables rules when the quota has been exceeded.

For now, this works well. I'm sure many improvements can be made and I'd like to get rid of the Python script and the cron job. I'm sure there are ways of limiting download speed using only iptables that I haven't found yet but it was a fun evening.

*As a last note: If you have IPv6 connectivity at home, make sure to duplicate all these rules in ip6tables as well. Youtube works fine over IPv6*