## Overview ##

nginx module to generate secure download links that can be used in combination with the module [ngx_http_secure_download](http://wiki.nginx.org/NginxHttpSecureDownload "ngx_http_secure_download"). 

The ngx_http_generate_secure_download_links module should be called via SSI, so it can replace SSI tags by secured download links on the fly like an output filter.

## Parameters ##

### generate_secure_download_link ###

Parameters: on/off

Default: off

Enable/disable the module for this location

### generate_secure_download_link_json ###

Parameters: on/off

Default: off

Add \ in front of all /, like json requires it

### generate_secure_download_link_url ###

Parameters: string

Default: none \(required\)

The url that should be secured

### generate_secure_download_link_expiration_time ###

Parameters: int

Default: none \(required\)

Time in seconds until the generated link should not be valid anymore

### generate_secure_download_link_period_length ###

Parameters: int

Default: 1

Time in seconds which specifies how often the links should be regenerated. This also influences the precision of the spcified expiration time.

### generate_secure_download_link_secret ###

Parameters: string

Default: none \(required\)

A string which should act as some kind of salt in the MD5 hash. It can also contain variables like the $remote_addr.

## Howto by example ##

### simple example ###

This is a very simple example which is making the Nginx generate links that expire after 1 hour. In the location / its important that SSI is turned on, because the generate_secure_download_links_module is access via the SSI. 

	37         location / {
	38             ssi on;
	39             root   html;
	40         }
	41 
	42        location /gen_sec_link {
	43             internal;
	44             rewrite /gen_sec_link(.*)$ $1 break;
	45             generate_secure_download_link_expiration_time 3600;
	46             generate_secure_download_link_secret MySecret$remote_addr;
	47             generate_secure_download_link_url $uri;
	48             generate_secure_download_link on;
	49         }

In the location / you could now have an html file like this.

	1 this is some text
	2 <a href="<!--# include virtual="/gen_sec_link/this_is_my_link" -->">this_is_my_link</a>
	3 some more text
	4 <a href="http://somewhateverhost.com<!--# include virtual="/gen_sec_link/this_is_another_link" -->">this_is_another_link</a>
	5 even more text

The Nginx SSI module will see the \<!\-\-\# include \-\-\> tag and replace it by what it gets from the URL which is specified in the virtual parameter "/gen_sec_link/this_is_another_link". So the Nginx will do an internal subrequest to the given url which is then going to the generate_secure_download_links module, the reply from that subrequest will replace the whole \<!\-\-\# include \-\-\>. The result will then look like for example this:

	1 this is some text
	2 <a href="/this_is_my_link/509325bc5fac6e4e42687fe096d67a9d/4C4EC7C3">this_is_my_link</a>
	3 some more text
	4 <a href="http://somewhateverhost.com/this_is_another_link/badbcb4d20500cca464c609da41001b2/4C4EC7C3">this_is_another_link</a>
	5 even more text

### JSON ###

One problem that i met is that in some cases you might want to output the secured link in json, which requires / to be escaped by \\. You could already generate an escaped string on the backend and put it into the virtual parameter of the SSI tag, which will then be secured by the Nginx. But the problem is that like this the \\ will be included in the MD5 hash. So when you then actually go to this page, you or the browser will take the \\ away so then the MD5 hash does not match anymore. 

What you have to do is to output an unescaped link in the backend, but tell the Nginx to do the JSON escaping, using the parameter generate_secure_download_link_json in the Nginx conf.

	42        location /gen_sec_link_json {
	43             internal;
	44             rewrite /gen_sec_link_json(.*)$ $1 break;
	45             generate_secure_download_link_expiration_time 3600;
	46             generate_secure_download_link_secret MySecret$remote_addr;
	47             generate_secure_download_link_url $uri;
	48             generate_secure_download_link on;
	49             generate_secure_download_link_json on;
	50         }

If your backend generates a JSON like for example this:

	{"preview_html":"style=\"background: url(http:\/\/bilder.lab<!--# include virtual='/gen_sec_link_json/9/8/C/28904_300.jpg' -->) no-repeat;\"

The output might look like this:

	{"preview_html":"style=\"background: url(http:\/\/bilder.lab\/gen_sec_link_json\/9\/8\/C\/28904_300.jpg\/badbcb4d20500cca464c609da41001b2\/4C4EC7C3) no-repeat;\"

So there you have a valid url with the \\ in front of each /.

### Periodical expiration times ###

A problem that I first ran into was that the browser caches don't work if the link changes every second, because it contains the timestamp. The solution for this problem was to introduce periodically generated expiration times. To enable this you can specify an amount of seconds in the parameter generate_secure_download_link_period_length.

	42        location /gen_sec_link {
	43             internal;
	44             rewrite /gen_sec_link(.*)$ $1 break;
	45             generate_secure_download_link_expiration_time 300;
	46             generate_secure_download_link_secret MySecret$remote_addr;
	47             generate_secure_download_link_url $uri;
	48             generate_secure_download_link on;
	49             generate_secure_download_link_period_length 60;
	50         }

A period length of 60 means that the Nginx first does the calculation of the timestamp when the link expires like usual, but then does "expiration_time = (expiration_time / period_length) * period_length" which simply floors the result down to the last multiple of period_length. So if you specify expiration_time 300 seconds and period_length 60 seconds, then the link will be valid for something between 240 and 300 seconds. This also means that the link will be changing only every 60 seconds, which allows the browser cache to do its working during that timespan.