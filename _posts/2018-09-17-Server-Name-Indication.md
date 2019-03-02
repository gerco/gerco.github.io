---
layout: post
title: Server Name Indication
---

Historically, SSL servers could only install a single certificate per IP address. There was no way for a client to specify which domain name they were trying to connect to before completing the SSL negotiations. This effectively made shared hosting impossible while using SSL. In 2003, an extension was added to the protocol (Server Name Indication, SNI) to resolve this issue and allow a server to serve the correct certificate for the host name a client is expecting ([RFC 3546](https://www.ietf.org/rfc/rfc3546.txt)).

There is still some old software around that does not support SNI so servers still may have to deal with clients that do and do not support SNI. They do this by configuring one certificate for each domain name for clients that support SNI and a single certificate with an alternative name for each of the configured domains for clients that don't. For example: If a server is responsible for three domain names:

- domain1.com
- domain2.com
- domain3.com

There would be four certificates configured on that server. One for each of the above domain names and another one that contains all three of the domains as alternative names (SANs).

Issues with SNI will often appear as cases where some clients will be able to connect and some will not be able to without any clear indications as to why. When checking the certificate with a browser, the correct certificate will appear to be served because the browser supports SNI. In order to check which certificate is served with- and without SNI, use the following commands:

| Scenario      | Command |
| :------------ | :------ |
| *without* SNI | `openssl s_client -connect domain1.com:443` |
| *with* SNI    | `openssl s_client -connect domain1.com:443 -servername domain1.com` | 

OpenSSL will print out the server's certificate in base64 encoding. Copy that to a file and use `openssl x509 -in domain1.crt -text` to convert it to a more readable format in order to check the certificate contents.
