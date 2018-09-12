---
format: post
title: Anonymous user counting through DNS
---

Imagine the following scenario:
You are maintaining a free application and you have a fair number of users, but you don’t know exactly how many. Because you want to understand your user base a bit better, you would like to count the number of users. This is the situation I found myself in recently.


###Requirements
My requirements for a system like this were as follows:

* Must work through restrictive corporate proxies;
* Collect the minimum amount of data possible;
* Do not annoy users;
* Little application size increase;
* Count number of users as accurately as feasible.

I investigated a few options, most of those were unacceptable to me for various reasons. They would either bloat my application, collect too much data or cause issues with the user’s network configuration. A lot of my users are behind a corporate proxy and my application is implemented in Java. Those of you who are familiar with Java will know that HTTP requests will - more or less randomly - pop up a proxy authentication dialog. Since I didn’t want to annoy my users I needed another solution.

I came up with a solution that fulfills most of the above requirements:

* Works through proxies because the application never makes a request to any server outside the corporate network;
* Collects only a random unique id that gets created on application startup. No user data is included whatsoever, not even the user’s IP address.
* Cannot pop up authentication dialogs or cause application delays due to firewalls restricting access to outside;
* Does not require any new libraries since all the code is already built in to in every OS.

This is how it works:

* On application startup, the application creates a random uuid;
* The application then attempts to resolve $uuid.stats.domainname.tld through DNS
* The DNS zone file for domainname.tld specifies a custom DNS server for stats.domainname.tld.
* stats.domainname.tld is running a custom DNS server that logs queries to $uuid.stats.domainname.tld.

There you have it, just count the unique ids in the DNS server’s log file and you’re done. This even has the added bonus of working on machines that have no internet access for security reasons. Even though they cannot connect anywhere outside the corporate network, they mostly still can resolve DNS queries! Now on to the implementation, which is surprisingly simple:

### Client
In the client application, all you need to do is generate a unique id and resolve it. In Java, this can be done as follows:

```java
public void getLatestVersionNumber() {
  Hashtable<String, String> env = new Hashtable<String, String>();	
  env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.dns.DnsContextFactory");
  DirContext ctx = new InitialDirContext(env);
  Attributes attrs = ctx.getAttributes(uuid + ".stats.domainname.tld", new String[] { "TXT" }); 
  Attribute attr = attrs.get("TXT");
  return attr.get().toString();
}
```

In this case, I’m retrieving the TXT record, since that will allow my custom DNS server to be able return some information to the user, like the most recent version of the application (to alert the user an update is available, for example).

###DNS configuration
The client code above causes the application to send a DNS request to resolve the domain name asked for: $uuid.stats.domainname.tld. In order to resolve this, the operating system will first need to resolve (in order): tld, dominate.tld and finally stats.domainname.tld. We can’t (or don’t want to) control which DNS server serves .tld or domainname.tld so we enter the following records to direct queries for *.stats.domainname.tld to our custom server:

```
; Set nameserver for stats.domainname.tld. to stats-ns.domainname.tld
stats     IN     NS     stats-ns

; Set IP address for stats-ns.domainname.tld. to 1.2.3.4
stats-ns  IN     A     1.2.3.4
```

Replace 1.2.3.4 with the IP address of the machine that you will be running your custom DNS server on. You will need root-access to that server since a DNS server must run on port 53. A shared hosting server will not work.

###DNS Server
On to the meat of the matter. Since I like experimenting with programming languages and my current language-du-jour is [Go](http://golang.org), I’m implementing this server in Go. I used the excellent [dns](https://github.com/miekg/dns) library from [Miek Gieben](http://miek.nl). The server is based on his “reflect” example (with most of the code removed). The only interesting part is below:

```go
func (h DNSHandler) ServeDNS(w dns.ResponseWriter, r *dns.Msg) {
     m := new(dns.Msg)
     m.SetReply(r)

     for _, q := range r.Question {
          // We only respond to TXT queries. Anything else does not exist on this DNS server
          if q.Qtype == dns.TypeTXT {
               var responseText string

               if strings.HasSuffix(q.Name, "."+h.config.DomainName) {
                    uuid := q.Name[:strings.Index(q.Name, ".")]
                    c := *newCheckIn(uuid, time.Now())
                    saveCheckin(c)

                    responseText = getLatestApplicationVersion()
               } else {
                    // The query is for config.DomainName, reply nothing
               }

               if len(responseText) > 0 {
                    t := new(dns.TXT)
                    t.Hdr = dns.RR_Header{Name: q.Name, Rrtype: dns.TypeTXT, Class: dns.ClassINET, Ttl: 3600}
                    t.Txt = []string{responseText}
                    m.Answer = append(m.Answer, t)
               }
          }
     }

     w.WriteMsg(m)
}
```

This simplified code stores the unique ids received and replies with the latest application version. 

That's all there is to it! Completely anonymous user counting without collecting any personal data, not even the user’s IP address. Naturally this only counts instances of the application and not the number of humans using it, but this is as close as you’re likely going to get.