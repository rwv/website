---
title: log (Caddyfile directive)
---

<script>
window.$(function() {
	// We'll add links to all the subdirectives if a matching anchor tag is found on the page.
	addLinksToSubdirectives();
});
</script>

# log

Enables and configures HTTP request logging (also known as access logs).

<aside class="tip">

If you're looking to configure Caddy's runtime logs, you're looking for the [`log` global option](/docs/caddyfile/options#log) instead.

</aside>


The `log` directive applies to the host/port of the site block it appears in, not any other part of the site address (e.g. path).

When configured, by default all requests to the site will be logged. To conditionally skip some requests from logging, use the [`skip_log` directive](/docs/caddyfile/directives/skip_log).


- [Syntax](#syntax)
- [Output modules](#output-modules)
  - [stderr](#stderr)
  - [stdout](#stdout)
  - [discard](#discard)
  - [file](#file)
  - [net](#net)
- [Format modules](#format-modules)
  - [console](#console)
  - [json](#json)
  - [filter](#filter)
    - [delete](#delete)
	- [rename](#rename)
	- [replace](#replace)
	- [ip_mask](#ip-mask)
	- [query](#query)
	- [cookie](#cookie)
	- [regexp](#regexp)
	- [hash](#hash)
- [Examples](#examples)

Since Caddy v2.5, by default, headers with potentially sensitive information (`Cookie`, `Set-Cookie`, `Authorization` and `Proxy-Authorization`) will be logged with empty values. This behaviour can be disabled with the [`log_credentials`](/docs/caddyfile/options#log-credentials) global server option.


## Syntax

```caddy-d
log [<logger_name>] {
	hostnames <hostnames...>
	output <writer_module> ...
	format <encoder_module> ...
	level  <level>
}
```

- **logger_name** is an optional override of the logger name for this site. By default, a logger name is generated automatically, e.g. `log0`, `log1`, and so on depending on the order of the sites in the Caddyfile. This is only useful if you wish to reliably refer to the output of this logger from another logger defined in global options. See [an example](#multiple-outputs) below.

- **hostnames** overrides the hostnames that this logger applies to. By default, the logger applies to the hostnames of the site block it appears in, i.e. the site addresses. This is useful if you wish to define different loggers per subdomain in a [wildcard site block](/docs/caddyfile/patterns#wildcard-certificates). See [an example](#wildcard-logs) below.

- **output** configures where to write the logs. See [Output modules](#output-modules) below. Default: `stderr`.

- **format** describes how to encode, or format, the logs. See [Format modules](#format-modules) below. Default: `console` if `stderr` is detected to be a terminal, `json` otherwise.

- **level** is the minimum entry level to log. Default: `INFO`. Note that access logs currently only emit `INFO` and `ERROR` level logs.



### Output modules

The **output** subdirective lets you customize where logs get written. It appears within a `log` block.

#### stderr

Standard error (console, default).

```caddy-d
output stderr
```

#### stdout

Standard output (console).

```caddy-d
output stdout
```

#### discard

No output.

```caddy-d
output discard
```

#### file

A file. By default, log files are rotated ("rolled") to prevent disk space exhaustion.

```caddy-d
output file <filename> {
	roll_disabled
	roll_size     <size>
	roll_uncompressed
	roll_local_time
	roll_keep     <num>
	roll_keep_for <days>
}
```

- **&lt;filename&gt;** is the path to the log file.
- **roll_disabled** disables log rolling. This can lead to disk space depletion, so only use this if your log files are maintained some other way.
- **roll_size** is the size at which to roll the log file. The current implementation supports megabyte resolution; fractional values are rounded up to the next whole megabyte. For example, `1.1MiB` is rounded up to `2MiB`. Default: `100MiB`
- **roll_uncompressed** turns off gzip log compression. Default: gzip compression is enabled.
- **roll_local_time** sets the rolling to use local timestamps in filenames. Default: uses UTC time.
- **roll_keep** is how many log files to keep before deleting the oldest ones. Default: `10`
- **roll_keep_for** is how long to keep rolled files as a [duration string](/docs/conventions#durations). The current implementation supports day resolution; fractional values are rounded up to the next whole day. For example, `36h` (1.5 days) is rounded up to `48h` (2 days). Default: `2160h` (90 days)


#### net

A network socket. If the socket goes down, it will dump logs to stderr while it attempts to reconnect.

```caddy-d
output net <address> {
	dial_timeout <duration>
	soft_start
}
```

- **&lt;address&gt;** is the [address](/docs/conventions#network-addresses) to write logs to.
- **dial_timeout** is how long to wait for a successful connection to the log socket. Log emissions may be blocked for up to this long if the socket goes down.
- **soft_start** will ignore errors when connecting to the socket, allowing you to load your config even if the remote log service is down. Logs will be emitted to stderr instead.



### Format modules

The **format** subdirective lets you customize how logs get encoded (formatted). It appears within a `log` block.

<aside class="tip">

**A note about Common Log Format (CLF):** CLF clashes with modern structured logs. To transform your access logs into the deprecated Common Log Format, please use the [`transform-encoder` plugin <img src="/resources/images/external-link.svg" class="external-link">](https://github.com/caddyserver/transform-encoder).

</aside>


In addition to the syntax for each individual encoder, these common properties can be set on most encoders:

```caddy-d
format <encoder_module> {
	message_key     <key>
	level_key       <key>
	time_key        <key>
	name_key        <key>
	caller_key      <key>
	stacktrace_key  <key>
	line_ending     <char>
	time_format     <format>
	time_local
	duration_format <format>
	level_format    <format>
}
```

- **message_key** The key for the message field of the log entry. Default: `msg`
- **level_key** The key for the level field of the log entry. Default: `level`
- **time_key** The key for the time field of the log entry. Default: `ts`
- **name_key** The key for the name field of the log entry (i.e. the name of the logger itself). Default: `name`
- **caller_key** The key for the caller field of the log entry.
- **stacktrace_key** The key for the stacktrace field of the log entry.
- **line_ending** The line endings to use.
- **time_format** The format for timestamps. May be one of:
  - **unix_seconds_float** Floating-point number of seconds since the Unix epoch; this is the default.
  - **unix_milli_float** Floating-point number of milliseconds since the Unix epoch.
  - **unix_nano** Integer number of nanoseconds since the Unix epoch.
  - **iso8601** Example: `2006-01-02T15:04:05.000Z0700`
  - **rfc3339** Example: `2006-01-02T15:04:05Z07:00`
  - **rfc3339_nano** Example: `2006-01-02T15:04:05.999999999Z07:00`
  - **wall** Example: `2006/01/02 15:04:05`
  - **wall_milli** Example: `2006/01/02 15:04:05.000`
  - **wall_nano** Example: `2006/01/02 15:04:05.000000000`
  - **common_log** Example: `02/Jan/2006:15:04:05 -0700`
  - Or, any compatible time layout string; see the [Go documentation](https://pkg.go.dev/time#pkg-constants) for full details.
- **time_local** Logs with the local system time rather than the default of UTC time.
- **duration_format** The format for durations. May be one of:
  - **seconds** Floating-point number of seconds elapsed; this is the default.
  - **nano** Integer number of nanoseconds elapsed.
  - **string** Using Go's built-in string format, for example `1m32.05s` or `6.31ms`.
- **level_format** The format for levels. May be one of:
  - **lower** Lowercase; this is the default.
  - **upper** Uppercase.
  - **color** Uppercase, with console colors.
  

#### console

The console encoder formats the log entry for human readability while preserving some structure.

```caddy-d
format console
```

#### json

Formats each log entry as a JSON object.

```caddy-d
format json
```


#### filter

Wraps another encoder module, allowing per-field filtering.

```caddy-d
format filter {
	wrap <encode_module> ...
	fields {
		<field> <filter> ...
	}
}
```

Nested fields can be referenced by representing a layer of nesting with `>`. In other words, for an object like `{"a":{"b":0}}`, the inner field can be referenced as `a>b`.

The following fields are fundamental to the log and cannot be filtered because they are added by the underlying logging library as special cases: `ts`, `level`, `logger`, and `msg`.

These are the available filters:

##### delete

Marks a field to be skipped from being encoded.

```caddy-d
<field> delete
```

##### rename

Rename the key of a log field.

```caddy-d
<field> rename <key>
```

##### replace

Marks a field to be replaced with the provided string at encoding time.

```caddy-d
<field> replace <replacement>
```

##### ip_mask

Masks IP addresses in the field using a CIDR mask, i.e. the number of bits from the IP to retain, starting from the left side. If the field is an array of strings (e.g. HTTP headers), each value in the array is masked. The value may be a comma separated string of IP addresses.

There is separate configuration for IPv4 and IPv6 addresses, since they have a different total number of bits.

Most commonly, the fields to filter would be `request>remote_ip` for the directly connecting client, `request>client_ip` for the parsed "real client" when [`trusted_proxies`](/docs/caddyfile/options#trusted-proxies) is configured, or `request>headers>X-Forwarded-For` if behind a reverse proxy.

```caddy-d
<field> ip_mask {
	ipv4 <cidr>
	ipv6 <cidr>
}
```

##### query

Marks a field to have one or more actions performed, to manipulate the query part of a URL field. Most commonly, the field to filter would be `request>uri`. The available actions are:

```caddy-d
<field> query {
	delete  <key>
	replace <key> <replacement>
	hash    <key>
}
```

- **delete** removes the given key from the query.
- **replace** replaces the value of the given query key with **replacement**. Useful to insert a redaction placeholder; you'll see that the query key was in the URL, but the value is hidden.
- **hash** replaces the value of the given query key with the first 4 bytes of the SHA-256 hash of the value, lowercase hexadecimal. Useful to obscure the value if it's sensitive, while being able to notice whether each request had a different value.

##### cookie

Marks a field to have one or more actions performed, to manipulate a `Cookie` HTTP header's value. Most commonly, the field to filter would be `request>headers>Cookie`. The available actions are:

```caddy-d
<field> cookie {
	delete  <name>
	replace <name> <replacement>
	hash    <name>
}
```

- **delete** removes the given cookie by name from the header.
- **replace** replaces the value of the given cookie with **replacement**. Useful to insert a redaction placeholder; you'll see that the cookie was in the header, but the value is hidden.
- **hash** replaces the value of the given cookie with the first 4 bytes of the SHA-256 hash of the value, lowercase hexadecimal. Useful to obscure the value if it's sensitive, while being able to notice whether each request had a different value.

If many actions are defined for the same cookie name, only the first action will be applied.

##### regexp

Marks a field to have a regular expression replacement applied at encoding time. If the field is an array of strings (e.g. HTTP headers), each value in the array has replacements applied.

```caddy-d
<field> regexp <pattern> <replacement>
```

The regular expression language used is RE2, included in Go. See the [RE2 syntax reference](https://github.com/google/re2/wiki/Syntax) and the [Go regexp syntax overview](https://pkg.go.dev/regexp/syntax).

In the replacement string, capture groups can be referenced with `${group}` where `group` is either the name or number of the capture group in the expression. Capture group `0` is the full regexp match, `1` is the first capture group, `2` is the second capture group, and so on.

##### hash

Marks a field to be replaced with the first 4 bytes (8 hex characters) of the SHA-256 hash of the value at encoding time. If the field is a string array (e.g. HTTP headers), each value in the array is hashed.

Useful to obscure the value if it's sensitive, while being able to notice whether each request had a different value.

```caddy-d
<field> hash
```



## Examples

Enable access logging to the default logger.

In other words, by default this logs to the console or stderr, but this can be changed by reconfiguring the default logger with the [`log` global option](/docs/caddyfile/options#log):

```caddy-d
log
```


Write logs to a file (with log rolling, which is enabled by default):

```caddy-d
log {
	output file /var/log/access.log
}
```


Customize log rolling:

```caddy-d
log {
	output file /var/log/access.log {
		roll_size 1gb
		roll_keep 5
		roll_keep_for 720h
	}
}
```


Delete the `User-Agent` request header from the logs:

```caddy-d
log {
	format filter {
		wrap console
		fields {
			request>headers>User-Agent delete
		}
	}
}
```


Redact multiple sensitive cookies. (Note that some sensitive headers are logged with empty values by default; see the [`log_credentials` global option](/docs/caddyfile/options#log-credentials) to enable logging `Cookie` header values):

```caddy-d
log {
	format filter {
		wrap console
		fields {
			request>headers>Cookie cookie {
				replace session REDACTED
				delete secret
			}
		}
	}
}
```


Mask the remote address from the request, keeping the first 16 bits (i.e. 255.255.0.0) for IPv4 addresses, and the first 32 bits from IPv6 addresses.

Note that as of Caddy v2.7, both `remote_ip` and `client_ip` are logged, where `client_ip` is the "real IP" when [`trusted_proxies`](/docs/caddyfile/options#trusted-proxies) is configured:

```caddy-d
log {
	format filter {
		wrap console
		fields {
			request>remote_ip ip_mask {
				ipv4 16
				ipv6 32
			}
		}
	}
}
```


<span id="wildcard-logs" /> To write separate log files for each subdomain in a [wildcard site block](/docs/caddyfile/patterns#wildcard-certificates), by overriding `hostnames` for each logger:

```caddy
*.example.com {
	log {
		hostnames foo.example.com
		output file /var/log/foo.example.com.log
	}
	@foo host foo.example.com  
	handle @foo {
		respond "foo"
	}

	log {
		hostnames bar.example.com
		output file /var/log/bar.example.com.log
	}
	@bar host bar.example.com
	handle @bar {
		respond "bar"
	}
}
```

<span id="multiple-outputs" /> To write the access logs for a particular subdomain to two different files, with different formats (one with [`transform-encoder` plugin <img src="/resources/images/external-link.svg" class="external-link">](https://github.com/caddyserver/transform-encoder) and the other with [`json`](#json)). 

This works by overriding the logger name as `foo` in the site block, then including the access logs produced by that logger in the two loggers in global options with `include http.log.access.foo`:

```caddy
{
	log access-formatted {
		include http.log.access.foo
		output file /var/log/access-foo.log
		format transform "{common_log}"
	}

	log access-json {
		include http.log.access.foo
		output file /var/log/access-foo.json
		format json
	}
}

foo.example.com {
	log foo
}
```
