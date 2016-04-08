---
layout: post
title:  "A brief history of the referer header"
tage:   [technical]
date:   2016-02-17 11:22:33 +0200
---

The referer header. Misspelled and misused since its inception.<!--more--> 
Its typical use is thus: if I click on a link in a webpage that takes me to a different page, the referer header shows the landing page which page I came from. 
It's very useful for gathering statistics about reading habits, but also presents a potential security risk, if too much information is passed on. 

In the original [RFC (2616)](http://www.w3.org/Protocols/rfc2616/rfc2616-sec15.html), the specification lays out that

> "Clients SHOULD NOT include a Referer header field in a (non-secure) HTTP request if the referring page was transferred with a secure protocol"

That is, if the origin of the request was https, the referer header should not be present.

However, RFCs are not mandatory, and data is sometimes leaked. 
Facebook fell foul of this a little while ago, when it turned out that in some cases the userid of the originating page was being passed in the referer header to advertisers when a user clicked on an advert. 
Their solution was to implement the [meta-referrer tag instead.](https://www.facebook.com/notes/facebook-engineering/protecting-privacy-with-referrers/392382738919)

The [meta-referrer tag](http://wiki.whatwg.org/wiki/Meta_referrer) can edit the referer header to allow sites to see where their traffic has come from, but without leaking potentially sensitive data.

Google were the first to [implement such a scheme](http://googlewebmastercentral.blogspot.co.uk/2012/03/upcoming-changes-inyy-googles-http.html), ostensibly to reducy latency from ssl sites, although one would suspect that being able to prove to clients that your site was the source of their traffic would be a more persuasive business argument.

The meta-referrer tag can take one of four values:

* never: omit the referer header from the request
* always: do not alter the referer header value, even if coming from an https page
* origin: set the referer header to be the document origin (ie not the full url)
* default: pass the full referer header when coming from http, but set it to be just the document origin when coming from https

Facebook, for example, use this tag so that clicks from their website to external sites indicate that the user came there from facebook, but without any of the information that might be leaked by the full url.
In the source code you can see

	<meta name="referrer" content="default" id="meta_referrer" />

and even when I'm logged into facebook and on a url such as
	
	https://www.facebook.com/joebloggs?f=nref

when I click on a link and view the headers, the referer header is reduced to

	referer=www.facebook.com

thus preserving my privacy.


Empty when:

* user entered url in address bar
* user visited site from a bookmark
* move from https to http
* move from https to different https url
* security software (antivirus, firewall etc) strips from requests
* behind a proxy that strips from requests
* visit the site programatically (eg curl) without setting header


References
----------

http://www.w3.org/Protocols/rfc2616/rfc2616-sec15.html

http://wiki.whatwg.org/wiki/Meta_referrer

http://www.schemehostport.com/2011/11/referer-sic.html

https://www.facebook.com/notes/facebook-engineering/protecting-privacy-with-referrers/392382738919

http://smerity.com/articles/2013/where_did_all_the_http_referrers_go.html

http://googlewebmastercentral.blogspot.co.uk/2012/03/upcoming-changes-inyy-googles-http.html

https://bugzilla.mozilla.org/show_bug.cgi?id=704320
