# TA-sendcef

TA-sendcef provides a custom Splunk search command, `sendcef`, for creating and sending CEF-formatted events over TCP or UDP. The underlying command uses the Splunk Custom Search Command protocol, version 2.

The specification for CEF, Common Event Format, is available in [PDF](https://kc.mcafee.com/resources/sites/MCAFEE/content/live/CORP_KNOWLEDGEBASE/78000/KB78712/en_US/CEF_White_Paper_20100722.pdf) from McAfee.

## Supported Splunk Versions

The `sendcef` custom command will only work with 6.4.x or higher versions of Splunk. Version 2 of the Splunk Custom Search Command protocol was introduced as a technology preview with Splunk 6.4.

## System Requirements

The `sendcef` command is a binary application written in [Go](https://golang.org). The TA has binaries included for the following platforms:
* Darwin (OS X) 64-bit
* Linux 64-bit
* Windows 64-bit

## Installation

Use normal Splunk installation procedures to install the TA. This includes installation through the Splunk UI or "unzipping" and copying to `$SPLUNK_HOME/etc/apps`. The application is a Splunk .spl file, which can be renamed to .tgz and decompressed.

Released versions are available on GitHub: [`TA-sendcef`](https://github.com/triddell/TA-sendcef/releases)

## Configuration

Configuration of `sendcef` is made through a custom conf file, specifically `sendcef.conf`. This file *must* be available in `$SPLUNK_HOME/etc/apps/TA-sendcef/local`. The `local` directory will need to be created after installing the application.

In the `local` directory, create the `sendcef.conf` file.

Here is an example configuration file:

```
[general]
#logLevel = debug
#timeout = 5s

[ceftarget:tcp]
server = <your_server_ip>:10522
protocol = tcp

[ceftarget:udp]
server = <your_server_ip>:10523
protocol = udp
```

### General Stanza

The `[general]` stanza above is not required but it does include some commented configurations that can be used to modify the behavior of `sendcef`.

Valid values for `logLevel` include `debug`, `info`, `warn`, and `error`. The default is `warn`. Most log messages are either `debug` or `error`.

The default `timeout` is `10s` (ten seconds). Valid, but not necessarily good, values are available from the Go `time.ParseDuration` function [documentation](https://golang.org/pkg/time/#ParseDuration).

### "Target" Stanzas

The "target" stanzas can be named anything. The examples append `:tcp` and `:udp` but a stanza could just as easily be `[foobar]`.

Within a target stanza, it's pretty basic. Name the stanza what you want. Then add a `server` value which includes a hostname or IP address followed by a port number (separated by a colon). For `protocol`, choose either `tcp` or `udp`.

### Usage

`sendcef` is a custom Splunk search command, so it's usually appended to a Splunk search. Here's a full example of a search command using `sendcef`:

```
index=_internal sourcetype=splunkd ERROR 
| convert timeformat="%m-%d-%Y %H:%M:%S.%3N" ctime(_time) AS cef_time 
| eval cef_host = host, 
    cef_device_vendor = "Vendor", 
    cef_device_product = "Product", 
    cef_device_version = "Version", 
    cef_event_class = "Class", 
    cef_name = "Name", 
    cef_severity = "10", 
    cef_extension = "message, source" 
| sendcef ceftarget:tcp
```

The base search, `index=_internal sourcetype=splunkd ERROR`, is simple and should work on nearly any Splunk system. If the search returns either zero records or many thousands, you may want to refine it to a managable number for your initial testing.

The next portion of the search sets up all of the CEF record format fields:

```
| convert timeformat="%m-%d-%Y %H:%M:%S.%3N" ctime(_time) AS cef_time 
| eval cef_host = host, 
    cef_device_vendor = "Vendor", 
    cef_device_product = "Product", 
    cef_device_version = "Version", 
    cef_event_class = "Class", 
    cef_name = "Name", 
    cef_severity = "10", 
    cef_extension = "message, source" 
```

These should become self-explanatory once we look at a sample output record.

The final portion of the search, `| sendcef ceftarget:tcp`, is what calls the `sendcef` custom search command. A configured stanza name is the sole command argument.

Here is an example record that would get sent to the target host and port over either TCP or UDP:

```
06-05-2017 10:09:26.392 Tims-MBP CEF:0|Vendor|Product|Version|Class|Name|10|message=File too small to check seekcrc, probably truncated.  Will re-read entire file\='/Users/tim/opt/splunk-sendtest/var/log/splunk/django_error.log'. source=/Users/tim/opt/splunk-sendtest/var/log/splunk/splunkd.log
```

In the search command, notice how full control of the event formatting is handled by using `convert` and `eval` to set key fields prior to calling `sendcef`. The full power of the Splunk Search Processing Language (SPL) is available to set these fields.

`sendcef` uses these fields when processing:
* cef_time 
* cef_host
* cef_device_product
* cef_device_vendor
* cef_event_class
* cef_name
* cef_severity
* cef_extension

This example hard-codes the CEF header fields but these values could just as easily be dynamic and based on the underlying Splunk event (using any SPL commands.)

The `cef_extension` field is handled differently. `cef_extension` takes a comma-separated list of fields from the underlying Splunk event and formats these name/value pairs (with escapes where necessary) before appending them to the end of the CEF message. Notice in the example output message above how `message` and `source` are handled as CEF extension fields.

**Note:** Don't rely on the order of these extension fields. `sendcef` randomizes the order of these extension fields in the output.

`sendcef` can be used as an ad hoc search command but it will often be used to automate data integration with third-party servers using Splunk saved/scheduled searches.

**Note:** The output of a command using `sendcef` will return the *unformatted, original* events to the browser as search results. A future version of `sendcef` may return the formatted CEF messages to the browser. Also, this custom command only *asks* Splunk for the `cef_*` fields and any fields within `cef_extension`. This could mean that additional extracted fields are not available in the search results.

### Testing

To faciliate testing using a local Splunk instance, a related TA has been developed. `TA-sendtest` provides test target indexes, TCP and UDP Splunk inputs, and a `sendcef.conf.sample` similar to what was used in this documentation.

Further information on this application is available on GitHub: [`TA-sendtest`](https://github.com/triddell/TA-sendtest)

### Troubleshooting

`sendcef` logs events at: `$SPLUNK_HOME/var/log/splunk/sendcef.log`

These events can be searched with Splunk using: `index=_internal source=*sendcef.log`

### Related Application

A similar application has been developed for sending "raw" Splunk events to third-party servers. Further information on this application is available on GitHub: [`TA-sendraw`](https://github.com/triddell/TA-sendraw)