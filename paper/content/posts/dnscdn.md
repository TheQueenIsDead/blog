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

### ⛅ DNS Providers

There's a few [DNS record types](https://en.wikipedia.org/wiki/List_of_DNS_record_types#Resource_records) out there, 
but TXT are expected to be the largest of them. The maximum size of attached data differs greatly between providers,
and isusually enforced via the provider API (SDK and UI).

A non-comprehensive overview of a few providers below demonstraces the variablity in content length. It is also important to
consider the significant difference in records that one can provision:

| Provider   | Max Length                                                                                                          | Max Records per Domain                                                          | Total Characters |
|------------|---------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|------------------|
| AWS        | [4,000 characters](https://aws.amazon.com/premiumsupport/knowledge-center/route-53-configure-long-spf-txt-records/) | [10,000](https://aws.amazon.com/route53/faqs/)                                  | 40,000,000       |
| Azure      | [1024 characters](https://learn.microsoft.com/en-us/azure/dns/dns-zones-records#txt-records)                        | [10,000](https://learn.microsoft.com/en-us/azure/dns/dns-zones-records#limits)  | 10,240,000       |
| Cloudflare | [2048 characters](https://developers.cloudflare.com/dns/manage-dns-records/reference/dns-record-types/#txt)         | 1,000 (Free)                                                                    | 2,048,000        |
| Namecheap | [2,500 characters](https://www.namecheap.com/support/knowledgebase/article.aspx/10058/10/namecheap-dns-limits/) | 150 | 375,000 |
The above values are based off the the respective "basic tier" for each provider. It also assumes that no smarts are performed with various 'record set' abstractions offered by some providers (groups of records with the same name).

Cost is a factor to consider that I have left out, and for many providers there may be monthly charges, or even billing per DNS query. 
While Cloudflare does not offer the largest amount of records, it doesn't tack-on extra charges at every chance it gets, 
and it's registrar provides domains at cost.

### 🌐 Internet

As well as provider specific limits, there is the general internet to think about as well.

[RFC4408](https://www.rfc-editor.org/rfc/rfc4408#section-3.1.3) as specified by [IETF](https://www.ietf.org/rfc/rfc4408.txt)
states that records can contain multiple strings. These are split into blocks of 255-bytes, which helps us exceed the maximum length of a single string within a DNS response.

However, item `3.1.4` states that record sizes in total should remain under 450 characters to allow the entire response to fit the 
size of a single [UDP packet](https://erg.abdn.ac.uk/users/gorry/course/inet-pages/udp.html), and results should remain 
within 512 octets in order to preserve compatibility with older infrastructure.

The wording "older infrastructure" implies that modern infrastrustre is happy to handle large records. 
So with that duly noted and ignored, let's deploy melvin.


## ⚒ Development 

I decided that it would be necessary to store some information about the media before retrieving it, an 'index' TXT record or sorts, that provides the count of subdomains to visit. accepting thata naming convention could be used to link them.

Thus the process for upload and download looks like so:

1. Upload `file.extension` @ `domain`
2. The CLI base64 encodes the data and splits it into blocks based on the max record size.
3. An index record is created at `<file.extension>.media.<domain>` with an integer representation of the amount of block domains.
4. Then for every block a data domain is created at `<file.extension>.<i>.media.<domain>`.

1. Download `file.extension` @ `domain`
2. The CLI retrieves the index record, then rtetrieves the TXT response of all data records.
3. The CLI pieces together the responses, base64 decodes it, and writes to disk.

This results in a list of domains that looks roughly like this:

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
 - Tests
 - CI (Lint, Test, Release)
 - Reduce code-duplication
 - Standardise logging

And in terms of functionality, depending on how much my wallet can bear:
 - Enumerate and list files on a domain
 - Support other DNS providers
   - Detect the provider automatically
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
