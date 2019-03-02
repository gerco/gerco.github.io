---
layout: post
title: NGINX reverse proxy with TLS passthrough
---

Like many other developers and tinkerers, I run a few services on my home server that I would like to access from the public internet when I'm not home. A good way to host many services on a single IPv4 address is to employ a reverse proxy, I use [NGINX](https://www.nginx.com) for that purpose. A combination of NGINX and a few shell scripts results in a system with automatically updating TLS certificates using [Let's Encrypt](https://letsencrypt.org), hosting services under many virtual (sub)domains and URLs. For example https://cloud.myname.com, https://home.myname.com/someservice, etc.

In the last few years, more and more services now natively support Let's Encrypt and that makes it very easy to protect them with a TLS certificate. To use the built-in Let's Encrypt support for those services it's often necessary for them to be the only service on port 443. They manage their own keys and certificates so it's difficult to host more than one service on any IPv4 address.

NGINX has a convenient feature that allows proxied services to directly manage their own certificates without needing the proxy server to be aware of the encryption keys: streams. The feature can even be combined with the regular https proxy for maximum flexibility:

The relevant configuration is as follows:

```
stream {
        map $ssl_preread_server_name $name {
                pbx.mydomain.com  pbx;
                default           yxorp;
        }

        upstream pbx {
                server 10.1.2.3:443;
        }

        upstream yxorp {
                server 127.0.0.1:1443;
        }

        server {
                listen 443;
                listen [::]:443;
                proxy_pass $name;
                ssl_preread on;
        }
}
```

This is a hypothetical configuration for a [FreePBX](https://www.freepbx.org) server that's managing its own certificate and a [Nextcloud](https://nextcloud.com) instance where the certificate is managed by the reverse proxy server. The DNS A records for pbx.mydomain.com and cloud.mydomain.com are both pointing to the same IPv4 address.

The `stream` section reads the server name the TLS client wants to connect to and then decides to which `upstream` the connection will be forwarded. If the client wants to connect to "pbx.mydomain.com", the connection is proxied directly to the FreePBX server at 10.1.2.3. This allows the FreePBX server to terminate the TLS connection and be in complete control of its certificates and encryption keys.

```
server {
        listen 1443 ssl http2;

        server_name cloud.mydomain.com;

        ssl on;
        ssl_certificate /etc/ssl/acme/cloud.mydomain.com/fullchain.pem;
        ssl_certificate_key /etc/ssl/acme/private/cloud.mydomain.com/privkey.pem;
        
        location / {
                proxy_pass http://10.2.3.4/;
        }
}
```

If the client wants to connect to a different domain name, the connection is proxied to localhost:1443 where the second configuration comes in. All the `server` sections that would normally be defined to listen on port 443 are listening to port 1443 instead. Since the `stream` server forwards all unknown domain names to this port, all domains configured as virtual hosts will work as normal.

```
server {
	listen		80;
	server_name	pbx.mydomain.com;

	location .well-known {
		proxy_pass http://10.1.2.3:80/.well-known;
	}
}
```

Lastly, the Let's Encrypt service needs to be able to access the `.well-known` directory on the FreePBX server for verification purposes. The FreePBX server will create a local file that Let's Encrypt will need to access. To make that possible, we define an extra `server` segment on port 80 for that hostname to ensure that the automatic domain validation can succeed.

This approach does have a serious downside: The applications behind the reverse proxy will lose the ability to determine the real IP address of their clients. All connections to the proxied services will be made by the reverse proxy so they will only see the proxy's address. This can normally be worked around by using the `X-Forwarded-For` header, but due to the use of the `stream` server as the point of entry, this is no longer possible.
