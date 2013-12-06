TimeSeries Format Specification v0.0-Proposal
=============================================

This proposal is issued by Steven Le Roux, <steven+ts@le-roux.info>. All contributions to improve it and lead up to a widely use of this format proposal are welcomed.

This proposal tries to summarize existing solutions in metrics collection and their respective message format. This work is mostly inspired by Statsd, Artimon, OpenTSDB and Sensision.

Since we are at the beginning of the TimeSeries era, the intent of this proposal is to try to federate a de facto standard, allowing a consistent growth of the software ecosystem arround TS (TimeSeries) like clients, aggregators, gateways, collectors, backends or analysis libraries.

Caveat
======

This proposal does not intend to be backward compliant with all existing tools, since some of them present missing capabilities or don't provide an extensible format.

This proposal covers only the representation of TimeSeries, which is mainly useful for emitters and restitution. It does not cover the data stucture used to implement a TS storage service.

Current metrics solutions
=========================

Data Aggregation
----------------

* *Statsd/Statsite* :
  Statsd, from the first Flickr perl sketch to the last Statsite C implementation via the well-known Etsy nodejs version, has done a great job pushing metrics from infrastructure like Codahale Metrics did for JVM based software. All Statsd based implementations provide an aggregation server which flushes to a backend like Graphite/Cube/Ganglia/... They all make use of a same Metrics Format : `key:value|type[|@flag]` which suffers from too restricted liberty (e.g. no tags support so you need to set the host in the key.

input format :
```
    key:value|type
    linux.proc.net.dc.rbx.rack.04.module.01.host.167837697.hostname.host_domain_tld.receive.bytes:123456789|c
```
output aggregated format :
```
    type.key|value|timestamp
    statsite.counts.linux.proc.net.dc.rbx.rack.04.module.01.host.167837697.hostname.host_domain_tld.dev.receive.bytes|123456789|1386208482
```
The inability to set tags turns the TS variable to be unreadable.

Data collection
---------------

* *Graphite/Ganglia* :
  These are both fixed size databases so the resulting collection will be slicked values. They does not support tags. Not really suitable for TS usage.

```
    metric_path value timestamp\n
    counts.linux.proc.net.dc.rbx.rack.04.module.01.host.167837697.hostname.host_domain_tld.dev.eth0.receive.bytes|123456789|1386208482
```


* *Cube* :
  is a frontend collector in front of MongoDB. Since it's a schemaless database, it ensures flexibility. On the other side, it's json serialization and more a MongoDB proxy than a truely TS analysis library. Its big flaw is the need for a request to know the TS key, instead of discovering them from tags.

``` 
    {
      "type": "linux.proc.net.dev.receive.bytes",
      "time": "2011-09-12T21:33:12Z",
      "tags": {
        "host": "10.1.0.1",
        "hostname": "host.domain.tld",
        "iface": "eth0",
        "dc": "rbx",
        "rack": "04",
        "module": "01"
      }
    }
```


* *OpenTSDB* :
  is a pure TS database with basic series manipulation. Compared to previous solution, it provides a tags support so that TS variables are the most generic possible. Unfortunatly, OpenTSDB has a 'just ok' readable TS format and does not support multi-dimensional TS.
```
    key timestamp value tags
    linux.proc.net.dev.receive.bytes 1386208482 123456789 host=10.1.0.1 hostname=host.domain.tld iface=eth0 dc=rbx rack=04 module=01
```


* *Artimon* :
  is the in-house TS framework at Crédit Mutuel Arkea. it offers a readable TS format with tags support, but is uni-dimensional..
```
    timestamp key{tags} value
    1386208482 linux.proc.net.dev.receive.bytes{host=10.1.0.1,hostname=host.domain.tld,iface=eth0,dc=rbx,rack=04,module=01} 123456789
```


* *Sensision* :
  is the heart TS software layer at CityzenData product. It supports tags and other items that makes it the most advanced message format for TimeSeries.
```
    timestamp// key{tags} value
    1386208482// linux.proc.net.dev.receive.bytes{host=10.1.0.1,hostname=host.domain.tld,iface=eth0,dc=rbx,rack=04,module=01} 123456789
```


Generic TimeSeries Format Proposal
==================================

Covering all cases, this proposal is to make use of the Sensision TS format, which does not enforce too much overprint for uni-dimensional TS.

TS format proposal :

    timestamp// key{tags} value

* Timestamp is the number of ms from unix epoch.

* // future use - dimensional criterion 

* Key is UTF-8 and dotted encoded string

* Tags are a key=value list, comma separated

* Value types definitions are described below :

 - 64 bits signed integer
 - 64 bits IEEE754
 - Boolean bit
 - UTF-8 encoded string

Moreover :

* Strings, from keys, tags keys, tags values and values, are %encoded following URL Encoding.

* From values
 - URLs are encoded between simple quotes : `'http://host.domain.tld/resource'`
 - Booleans can be described as `T/F/t/f/true/false`
 - Doubles are in the form `-?[0-9]+\.[0-9]+` like 1.0001, 42.42 or -12.34. No use of scientific notation.


Resulting Series
================

This format offers the ability for a CLI to present a serie under the form for a given time range:

```
    health.heart.beats{unit=BeatsPerMin,freq=min} 60 65 68 72 75 85 89 91 94 96 99 102 105 100 95 91
    linux.proc.cpu.load{host=1.1.1.1,hostname=host.domain.tld,rack=01,position=u30} 1 0 0 0 0 1 2 3 4 10 3 2 1 1 1 0
    ipmi.fan.rpm{host=1.1.1.1,hostname=host.domain.tld,rack=01,position=u30} 200 200 200 200 200 201 202 200 200 203 205 350 500 1500 2500 1300 600 -
    ipmi.fan.status{host=1.1.1.1,hostname=host.domain.tld,rack=01,position=u30}} 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 -
    home.sensors.temperature{unit=celcius,roomId=10,roomName=parentsBedRoom,house=SF} 20 19 19 19 19 20 20 20 19 19 20 19 20 21 21 22 21 21 
    ...
```


Out of scope definitions
========================

Compressed and/or crypted messages are left to the Transport layer so that the TS format is agnostic for those aspects.

It offers to service providers the ability to provide differents optimizations related to a given context. This is why we're not proposing a protocol, just a format.

Left to Client/Server implementations :

  * Compression
  * Cryptography
  * Transport Protocol
    - binary/HTTP/Auth/...
  * Backend storage format


Contributors
============



References
==========

* [Flickr StatsD](http://code.flickr.com/blog/2008/10/27/counting-timing/)
* [Etsy StatsD](https://github.com/etsy/statsd)
* [Coda Hale's Metrics](http://metrics.codahale.com/)
* [Graphite](http://graphite.wikidot.com/)
* [OpenTSDB schema](http://opentsdb.net/schema.html)
* [Artimon](http://fr.slideshare.net/Mathias-Herberts/20111109-artimonapache-flumemeetupfinal2)
* [StatsD Metrics Export Specification ](https://github.com/b/statsd_spec)
* [URL Encoding](http://en.wikipedia.org/wiki/Percent-encoding)
* Private conversations with [CityzenData](http://cityzendata.com)
