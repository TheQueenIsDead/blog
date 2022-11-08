---
title: "DNSCDN"
date: 2022-11-08T18:07:06+13:00
draft: false
tags: ["dns", "cloudflare", "go", "golang", "cli"]
description: "🐈💾 Storing pictures of cats where they aren't supposed to be 🌐"
showToc: true
TocOpen: true
---

Another side project, this time with partial completion and a valid use (?) for messing with my domain!

**[TheQueenIsDead/dnscdn](https://github.com/TheQueenIsDead/dnscdn): Domain Name System Content Distribution Network.**

Affectionately known as MelvinDB.

![Melvin](https://raw.githubusercontent.com/TheQueenIsDead/dnscdn/main/melvin.png)

To the non-techy layman, this may sound like the next best technological marvel since scamming people with Web3,
but for those that have a little computer know-how, it invokes a more puzzled expression.

Per the project [README.md](https://github.com/TheQueenIsDead/dnscdn/blob/main/README.md), I wondered what the limits of
data within DNS records looks like and the theoretical limits for file storage.

Here's my findings!

## 📈 Limits 

### ⛅ Cloud Providers

There's a few [DNS record types](https://en.wikipedia.org/wiki/List_of_DNS_record_types#Resource_records) out there, 
but TXT are expected to be the largest of them. This is important as various provider SDK's are prone to limiting how
much data you can store, per the below chart:

| Provider   | Max Length                                                                                                          | Max Records per Domain                                                          | Total Characters |
|------------|---------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|------------------|
| AWS        | [4,000 characters](https://aws.amazon.com/premiumsupport/knowledge-center/route-53-configure-long-spf-txt-records/) | [10,000](https://aws.amazon.com/route53/faqs/)                                  | 40,000,000       |
| Azure      | [1024 characters](https://learn.microsoft.com/en-us/azure/dns/dns-zones-records#txt-records)                        | [10,000](https://learn.microsoft.com/en-us/azure/dns/dns-zones-records#limits)  | 10,240,000       |
| GCP        | ?                                                                                                                   |                                                                                 |                  |
| Cloudflare | [2048 characters](https://developers.cloudflare.com/dns/manage-dns-records/reference/dns-record-types/#txt)         | 1,000 (Free)                                                                    | 2,048,000        |

The above assumes the most basic public tier, and that no smarts are performed with various 'record set' abstractions utilised by cloud providers.
Google notes that record strings have maximum values but does not seem to indicate the maximum record size they would be willing to create.

It does also not take into account monthly charges or billing per DNS query. While Cloudflare does not offer the 
largest amount of records, it doesn't tack-on extra charges at every chance it gets, and it's registrar provides domains
at cost.

### 🌐 Internet

[RFC4408](https://www.rfc-editor.org/rfc/rfc4408#section-3.1.3) as specified by [IETF](https://www.ietf.org/rfc/rfc4408.txt)
states that records can contain multiple strings, which helps us exceed the 255-byte maximum string length of a DNS response.

However, item `3.1.4` states that record sizes should remain under 450 characters to allow the entire response to fit the 
size of a single [UDP packet](https://erg.abdn.ac.uk/users/gorry/course/inet-pages/udp.html), and results should remain 
within 512 octets in order to preserve compatibility with older infrastructure.

With that duly noted and ignored, let's deploy melvin.


## ⚒ Development 

As I worked through I realised that I had resorted to hard coding the amount of records to enumerate for `melvin.png`.
This could have been solved by reserving data from the end of the DNS record content, pointing to the next record to visit, 
or we could turn to our lord and saviour...DNS.

I decided to implement an 'index' TXT record that provides the count of subdomains to visit, accepting that
a naming convention could be used to link them.

This changed the process to:

1. Upload `file.extension`
2. An index record is stored at `<file.extension>.media.<domain>` with an integer representation of the amount of data domains.
3. The CLI then knows to iterate `<file.extension>.<i>.media.<domain>`, and when to stop.
4. The CLI pieces together the responses, deserializes it and write it to disk.

This results in a list that looks roughly like this:

```console
melvin.png.media.tqid.dev       =>  7
melvin.png.0.media.tqid.dev     => <base64 data>
melvin.png.1.media.tqid.dev     => ...
melvin.png.2.media.tqid.dev     => ...
melvin.png.3.media.tqid.dev     => ...
melvin.png.4.media.tqid.dev     => ...
melvin.png.5.media.tqid.dev     => ...
melvin.png.6.media.tqid.dev     => ...
```

It's a palatable solution to provide a working CLI until limits are encountered :-) 

## ⏳📃 TODO

With an MVP in place, the next steps from here would likely be spit and polish bits like:
 - tests
 - ci: lint, test, release
 - reduce code-duplication
 - standardise logging a bit more

And in terms of functionality, depending on how much my wallet can bear:
 - Add an AWS and Azure provider
 - Add a means to detect the provider automatically
 - Test with bigger datasets
   - Attempt to convert to async go routines for data fetching

## 🔚 Wrap-up

This works surprisingly well, and has been a great foray into Golang, especially the [urfave/cli](https://github.com/urfave/cli)
package that made converting my monolithic `main.go` into a CLI unfairly easy.

Feel free to try downloading melvin yourself by [cloning the repository](https://github.com/TheQueenIsDead/dnscdn) and running:

```console
dnscdn -f melvin.png -d tqid.dev download
```

Looking forward to more Golang usage, and adding in little tweaks here and there!

Thanks for the read ♥
