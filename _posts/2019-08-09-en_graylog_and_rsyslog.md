---
layout: default
title: Rsyslog and Graylog
description: How to use rsyslog as a source of GELF messages and not screw up
categories: [en, devops]
tags: [rsyslog, graylog, logging]
---

# Rsyslog и Graylog

Nowadays, You use modern log analytics systems like graylog, ELK, Splunk etc if you need to audit your log messages. And you need to audit your logs if you have more than a couple of servers or reliable monitored application. We are using Graylog in our company (matter of chance), so the rest part of article is a summary of my experience with it.

So, you installed graylog and need to feed it with log messages. There are few ways.

1. Server agents. They are reading log files, format them in a more reasonable form than plain string, and forward such message to graylog server. Beats and NXLog are such agents to name a few. They even can be centrally configured with graylog sidecars.

2. Any syslog capable server (rsyslog, syslog-ng etc) and parsing messages further with graylog server.

3. Send GELF messages directly to graylog.

Each of these ways has its own pros and cons. Let's talk about them in a bit more detail.

## Step 1. Rsyslog as plain syslog agent

Syslog was my first choice. Our debian servers already has rsyslog installed out of the box. Rsyslog can send encrypted syslog messages via internet. Graylog has regexes and grok-patterns. What could possibly go wrong?

Well, one thing definitely could. Even our small load (~40 rps) sometimes led to graylog hang-ups. Exactly like in this [issue](https://github.com/Graylog2/graylog2-server/issues/4326). Similar topics on graylog community forum suggest to add more nodes to graylog cluster. Or restart graylog service. Both options suck.

And another small issue kept bugging me too. Rsyslog sends multiline messages with _\\n_ (escaped LF) as a string delimiter. But there is no way to split the message back by _\\n_ symbol sequence at GrayLog side. Why? No idea. And the GrayLog team doesn't see it like a problem: [1](https://github.com/Graylog2/graylog2-server/issues/5971), [2](https://github.com/Graylog2/graylog2-server/issues/5225), [3](https://github.com/Graylog2/graylog2-server/issues/4536) and list of such issues is even longer.  

All this led us to the decision to format messages on the servers and send to GrayLog already preparred data.

## Step 2. Filebeats and NXLog Agents

So we started to experiment with agents and GrayLog sidecars. At that moment (year and half ago) nxlog has no deb package for Debian Stretch, so I investigated Filebeats. 
Beats is great if you need quick out of the box solution. And if you didn't modify log format for your services. Unortunately, that was my case. We use custom formats for Apache and Nginx logs. Probably, there is a way to enhance beats Nginx module, but I didn't find one. Also beats can't read data from unix socket, but only from file.

So I have not find any advantages of beats over rsyslog. And that was the argument to not introduce new dependency in a log chain.

## Step 3. Rsyslog as gelf forwarder

As I wrote before, Rsyslog was my first choice to forward log to Graylog (using syslog proto). It is installed in Debian out of the box, has good documentation. And It was well studied by me during first "plain syslog" attempt. And after the failure of "sidecar and beats" attempt I started to think - "may be there is a better way to use rsyslog as a forwarder to Graylog?".

### How It works

Rsyslog can't convert log messages to GELF out of the box - you need to create templates first. Next, you need to create a ruleset to forward data to your graylog server and implement the template. 

```
template(name="gelf" type="list") {
  constant(value="{\"version\":\"1.1\",")
  constant(value="\"host\":\"")                  property(name="hostname")
  constant(value="\",\"short_message\":\"")      property(name="msg" format="json")
  constant(value="\",\"timestamp\":")            property(name="timegenerated" dateformat="unixtimestamp")
  constant(value=",\"_application_name\":\"")    property(name="app-name")
  constant(value="\",\"_facility\":\"")          property(name="syslogfacility-text")
  constant(value="\",\"level\":\"")              property(name="syslogseverity")
  constant(value="\"}")
}

call graylog
ruleset(name="graylog") {
  action(
    type="omfwd"
    Target="graylog.example.com"
    Port="12201"
    Protocol="tcp"
    KeepAlive="on"
    template="gelf"
    StreamDriver="gtls"
    StreamDriverMode="1"
    StreamDriverAuthMode="x509/name"
    StreamDriverPermittedPeers="graylog.example.com"
    TCP_FrameDelimiter="0"
  )
}
```

This snippet is a slightly different from [rsyslog gelf guide](https://www.rsyslog.com/doc/master/tutorials/gelf_forwarding.html):

1. timestamp field is quoted in rsyslog docs. It's against current GELF standard, timestamp should be a number, not string.
2. There are two additional fields - *\_application\_name* and *\_facility*. They are easy to add and it makes our gelf messages more informative.
3. omfwd module uses TCP protocol instead of UDP. This is perhaps the most mysterious difference. Rsyslog doc says "_We now have a syslog forwarding action. This uses the omfwd module. Please note that the case above only works for UDP transport. When using TCP, Graylog expects a Nullbyte as message delimiter. This is currently not possible with rsyslog_". And it is false. Rsyslog supports nullbyte as a delimiter in omfwd [since 2017](https://github.com/rsyslog/rsyslog/issues/1602). See [omfwd module doc](https://www.rsyslog.com/doc/master/configuration/modules/omfwd.html#tcp-framedelimiter).
4. SSL certificates and gtls module are used to encrypt communication between rsyslog and Graylog. 

So, you can use rsyslog as gelf forwarder with modern Graylog installations (we are using graylog 2.5 now and have plans to upgrade to 3.1). With TCP, sessions and encryption. I created [PR](https://github.com/rsyslog/rsyslog-doc/pull/840) to Rsyslog documentation and it will published with a new Rsyslog release.

### Custom fields in GELF messages 

It is stupid to install Graylog but not use it for advanced analytics. Basic GELF message allows us to lookup errors (severity field), do full text search by message body, but that's it. We need custom fields in GELF message to do more sophisticated analysis. For example, to create map dashboard we need client IP address in custom field. Lucky for us, Rsyslog has [mmnormalize module](https://www.rsyslog.com/doc/master/configuration/modules/mmnormalize.html) and [liblognorm](https://github.com/rsyslog/liblognorm/blob/master/doc/configuration.rst).

Liblognorm is a library to parse log messages with a grok-like patterns, but without regexes. So it is faster. Mmnormalize is rsyslog module which binds liblognorm rulebases and rsyslog rulesets. But we need slightly different template for such GELF messages. I named it **gelf-ext**.

```
template(name="gelf-ext" type="list") {
  constant(value="{\"version\":\"1.1\",")
  constant(value="\"host\":\"")                  property(name="hostname")
  constant(value="\",\"short_message\":\"")      property(name="msg" format="json")
  constant(value="\",\"timestamp\":")            property(name="timegenerated" dateformat="unixtimestamp")
  constant(value=",\"_application_name \":\"")   property(name="app-name")
  constant(value="\",\"level\":\"")              property(name="syslogseverity")
  constant(value="\",")                          property(name="$!all-json" position.from="2")
}

ruleset(name="graylog-ext") {
  action(
    type="omfwd"
    Target="graylog.example.com"
    Port="12201"
    Protocol="tcp"
    KeepAlive="on"
    template="gelf-ext"
    StreamDriver="gtls"
    StreamDriverMode="1"
    StreamDriverAuthMode="x509/name"
    StreamDriverPermittedPeers="graylog.example.com"
    TCP_FrameDelimiter="0"
  )
}
```

The main difference between **gelf** and **gelf-ext** templates is **$!all-json** variable. It is json string created by liblognorm rulebase from original log message. And **graylog-ext** ruleset is used to forward **gelf-ext** messages to Graylog. 

And finally, I want to give your a few examples how to write rulebases and actions. Let's start with a simple example.

#### ProFTPD xferlog

Xferlog is ProFTPD server log file with a data about file modifications. You can read [here](https://linux.die.net/man/5/xferlog) about its format.

Example message: 

```
Sun Dec 30 01:38:57 2018 0 80.237.119.90 22062 /var/www/site/path/to/dir/file.php b _ o r username sftp 0 * c
```

We want to extract client\_ip, username, protocol and other fields from the message. Here is a liblognorm rulebase

```
version=2

rule=proftpd,info,gelf:%
  -:word
% %
  _timestamp:date-rfc3164{"format":"timestamp-unix"}
% %
  -:number
% %
  _transfer_time_:number{"format":"number"}
% %
  _client_ip:word
% %
  -:number
% %
  _file_name:word
% %
  _transfer_type:word
% %
  -:word
% %
  _direction:word
% %
  _access_mode:word
% %
  _user_name:word
% %
  _service_name:word
% %
  -:word
% %
  -:word
% %
  _completion_status:word
%
```
A few comments: 
* **word** parser is used to find any non-space sequence, **-** is a special field name to match something but not to save the match into final json. So **%-:word%** means "skip this word and go further".  
* **number{"format":"number"}** is a parser with parameter. By default, any extracted field will be string value in a final json. Even with a number parser. So we need to use format parameter to make the value numeric. It is useful, because we can use statistic functions (mean, max, min etc) only for numberic values. There is no such statistic for strings even if string looks like number.

And we need a few more rsyslog directives to bind our **graylog-ext** ruleset and this rulebase.

```
input(
  type="imfile"
  File="/var/log/proftpd/xferlog"
  Tag="proftpd"
  Severity="info"
  Facility="local7"
  ruleset="proftpd"
)

ruleset(name="proftpd") {
  action(type="mmnormalize" rulebase="/etc/rsyslog.d/rules/proftpd_xfer.rb")
  call graylog-ext
}
```

Let's move to more sophisticated example.

## PHP-FPM Slowlog

Slowlog is a multiline stacktrace message from PHP-FPM. 

Example message:

```
[02-Jan-2019 02:10:20]  [pool pool_name] pid 13776
script_filename = /var/www/site/path/to/dir/file.php
[0x00007fc26e017500] curl_exec() /var/www/site/path/to/dir/file.php:351
[0x00007fc26e017430] client() /var/www/site/www/local/widget/scripts/service.php:301
[0x00007fc26e0171f0] requestPV() /var/www/site/www/local/widget/scripts/service.php:148
[0x00007fc26e017160] getPVFile() /var/www/site/www/local/widget/scripts/service.php:34
[0x00007fc26e0170c0] getPV() /var/www/site/www/local/widget/scripts/service.php:11
```

Rulebase:
```
version=2

rule=php-fpm,info,gelf:\\n[%date:char-to:]%]  [pool %pool_name:char-to:]%] pid %-:number%\\nscript_filename = %script_filename:string-to{"extradata":"\\n"}%\\n%full_message:rest%
```

And rsyslog directives:

```
input(
  type="imfile"
  file="/var/www/*/logs/*log.slow"
  startmsg.regex="^$"
  tag="php-fpm"
  readTimeout="10"
  ruleset="php-fpm-slow"
  addMetadata="off"
)

ruleset(name="php-fpm-slow") {
  action(type="mmnormalize" rulebase="/etc/rsyslog.d/rules/php_fpm_slow.rb")
  set $!full_message = replace($!full_message, '\\n', "\n");
  call graylog-ext
}
```

Pay attention at string `set $!full_message = replace($!full_message, '\\n', "\n");`. We replace `\\n` (three symbol sequence) with a line feed symbol back again. And Graylog will show this message as multi-line.

Finally, the most tricky example.

## Nginx error_log

Nginx error log contains a different type of messages. Nginx internal errors, fastcgi errors and other types can be found here. Log messages can be both single-line and multi-line. And for the icing on the cake: messages has prefix, body and suffix. Moreover, suffix can vary from message to message.
 
Example messages:

```
 2019/08/07 11:01:52 [error] 22493#22493: *2117401 FastCGI sent in stderr: "PHP message: [2019-08-07 11:01:52] production.ERROR: Error Processing Request {"userId":2,"email":"user@domain.tld","exception":"[object] (Exception(code: 1): Error Processing Request at /var/www/site/www/app/Http/Controllers/HomeController.php:26)
[stacktrace]
#0 [internal function]: App\\Http\\Controllers\\HomeController->index()
#1 /var/www/site/www/vendor/laravel/framework/src/Illuminate/Routing/Controller.php(54): call_user_func_array(Array, Array)
#2 /var/www/site/www/vendor/laravel/framework/src/Illuminate/Routing/ControllerDispatcher.php(45): Illuminate\\Routing\\Controller->callAction('index', Array)
#3 /var/www/site/www/vendor/laravel/framework/src/Illuminate/Routing/Route.php(212): Illuminate\\Routing\\ControllerDispatcher->dispatch(Object(Illuminate\\Routing\\Route), Object(App\\Http\\Controllers\\HomeController), 'index')
#4 /var/www/site/www/vendor/laravel/framework/src/Illuminate/Routing/Route.php(169): Illuminate\\Routing\\Rout" while reading response header from upstream, client: 188.126.90.15, server: host, request: "GET / HTTP/2.0", upstream: "fastcgi://unix:/var/run/php/site.sock:", host: "site"
 24079#24079: *7858927 open() "/var/www/site/www/cache/js/s1/_catalog/page_a772ecd0357d0d28edb5730c1c1fdbfc/page_a772ecd0357d0d28edb5730c1c1fdbfc_v1.js,q15428907836424.pagespeed.jm.ICbv86Z6iJ.js" failed (2: No such file or directory), client: 66.249.66.54, server: site, request: "GET /cache/js/s1/_catalog/page_a772ecd0357d0d28edb5730c1c1fdbfc/page_a772ecd0357d0d28edb5730c1c1fdbfc_v1.js,q15428907836424.pagespeed.jm.ICbv86Z6iJ.js HTTP/1.1", host: "site", referrer: "https://site/catalog/path/to/dir/"
```

Rulebase:

```
version=2

type=@value:%..:char-to:,%
type=@value:"%..:char-to:"%"

prefix= %-:word% %-:word% [error] %-:number%#%-:number%: *%-:word% 

rule=gelf,monolog,nginx,error:FastCGI sent in stderr: "PHP message: [%-:char-to:]%] %full_message:string-to{"extradata":"\" while reading response header from upstream"}%" while reading response header from upstream, %
    {"name":"meta", "type":"repeat",
    "parser":[
               {"type":"char-to", "name":"key", "extradata":":"},
               {"type":"literal", "text":": "},
               {"type":"@value",  "name":"value"}
             ],
    "while":[
               {"type":"literal", "text":", "}
            ]
    }%

rule=gelf,nginx,error:%full_message:string-to{"extradata":", client:"}%, %
    {"name":"meta", "type":"repeat",
    "parser":[
               {"type":"char-to", "name":"key", "extradata":":"},
               {"type":"literal", "text":": "},
               {"type":"@value",  "name":"value"}
             ],
    "while":[
               {"type":"literal", "text":", "}
            ]
    }%
```
* 
* We created a new liblognorm user type here - **@value**. This is any char sequence ended with comma or double quote. етс запятой или двойной кавычкой. 
* Prefix liblognorm pragma is used to define common prefix for several rules. It helps to reduce rule length.
* Pay attention to [space symbol](https://www.rsyslog.com/log-normalization-and-the-leading-space/) at the start of the rule.
* And (my favourite) we used **repeat** parser. It can extract fields inside a loop. Nginx adds meta fields in the end of the message and the list of fields may vary. For example, not every request is processed by upstream server, so upstream may be omitted.

Every key and value extracted by repeat parser will be stored in **meta** key. JSON looks like this:
```
{
	"tags": ["gelf","nginx","error"],
	"full_message": "...",
	"meta": [
		{ "key": "client", "value": "..." },
		{ "key": "request", "value": "..." },
		...
	]
}
```

It's bad, because GELF standard allows only strings and numbers, not objects and arrays, as values. That's how we can solve it.

```
input(
  type="imuxsock"
  Socket="/run/nginx.syslog"
  ruleset="nginx"
)

ruleset(name="nginx-error-log") {
    action(type="mmnormalize" rulebase="/etc/rsyslog.d/rules/nginx_error.rb")
    set $!full_message = replace($!full_message, '#012', "\n");
    foreach ($.i in $!meta) do {
        if ($.i!key == 'host') then {
            set $!_vhost = $.i!value;
        } else if ($.i!key == 'client') then {
            set $!_client_ip = $.i!value;
        } else if ($.i!key == 'server') then {
            set $!_server = $.i!value;
        } else if ($.i!key == 'request') then {
            set $!_http_request = $.i!value;
        } else if ($.i!key == 'upstream') then {
            set $!_upstream = $.i!value;
        }
    }
    unset $!meta;
  }
  call graylog-ext
}
```

* Line feed symbol is encoded as _#012_, not _\\n_. Because data is read from the socket, not from file.
* [Rainer script](https://www.rsyslog.com/doc/master/rainerscript/index.html) loop is used to extract meta information from subkey and place it to the first level of $!all-json variable.

## Conclusion

Rsyslog is very powerful tool to parse and forward log messages and it can be smoothly used with Graylog. I hope, this article helps you to solve some problems in your infrastructure and to save some money with less hardware greedy technologies. 
