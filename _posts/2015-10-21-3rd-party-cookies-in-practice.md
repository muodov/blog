---
layout:     post
title:      3rd-party cookies in practice
summary:    Quick overview of the effects of disabling 3rd-party cookies in browser
categories: web
published:  true
comments:   true
---

Recent days I was building a cross-site communication tool.
So I was looking into [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS), [window.postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) and other relevant stuff.

While these mechanisms seem to be well-documented and have simple APIs,
handling 3rd-party cookies is a little bit vague. In particular, I was interested
in _What exactly does it mean when user disables 3rd-party cookies?_.

It turned out not so complicated, but you don't know this until you actually have
to use it, so hopefully this will help someone in future.

## Questions

First of all, what is a 3rd-party cookie anyway? Common sense tells us this must be
cookies in scope of domains other than the current page. But this is quite vague.
Does it mean we only cannot set new cookies, should we be able to read the old ones?
How should browser handle cookies set from Javascript? Does this rule affect requests
triggered by images and other resources? How does it affect CORS requests? What about localStorage and sessionStorage? And how do we detect that user has disabled 3rd-party cookies?

## Answers

I've done a quick testing trying to answer those questions and here is what I found.
Everything was checked on Firefox 41.0.2, Google Chrome 46.0.2490.71, and Safari 9.0 under Mac OS El Capitan.

* First of all, when 3rd-party cookies are disabled, browsers **_do not save, nor send_**
cookies for domains other than the top-level window (current page). This affects all cross-domain requests including resources (e.g. `<img>` tags), iframes, and XMLHttpRequests (including CORS).
This is something that you would expect, actually.
* If there is a cookie that was set previously as a 1st-party, it won't be used.
* Javascript in cross-origin iframe cannot set cookies as well. No exceptions
will be thrown, but `document.cookie` **_will always return an empty string_**, even
if you set it to something.
* No cookies in CORS requests too, even if you use `.withCredentials` parameter. Cookie
header from server is just ignored.

`localStorage`/`sessionStorage` is a bit more tricky:

* in Google Chrome any attempt to access `localStorage` will result in `DOMException`
* in Safari, `localStorage.getItem()` and `localStorage` itself doesn't raise errors, but
if you try to use `localStorage.setItem()`, it will give you a `QuotaExceededError`
* in Firefox, localStorage and sessionStorage are still working inside iframes.
So you can save stuff in localStorage, and it will stay there. However, this
[will be changed](https://bugzilla.mozilla.org/show_bug.cgi?id=536509) in upcoming Firefox 43.

## Detection

If you need to detect if 3rd-party cookies are enabled, one option is to take
advantage of always-empty document.cookie. Just create an iframe pointing to
a page on another domain like this:

{% highlight html %}
<html>
<head>
<script>
var randStr = function(){
  return Math.random().toString(36).substring(7);
};

var thirdPartyCookiesEnabled;
var k = randStr(),
    v = randStr();
document.cookie = k + '=' + v;
if(document.cookie.indexOf(k + '=' + v) == -1) {
  thirdPartyCookiesEnabled = false;
} else {
  thirdPartyCookiesEnabled = true;
}
alert(thirdPartyCookiesEnabled);
</script>
</head>
<body></body>
</html>
{% endhighlight %}

## Workaround

There is one workaround though. If you really need to access cookies on another domain,
just make sure that they are 1st-party. Essentially, it means **_redirects_**.
So you can redirect user to a page to another domain, set the cookie there, and then
immediately redirect him back. This is not the best user experience, but it still works.

Hope this will help someone.