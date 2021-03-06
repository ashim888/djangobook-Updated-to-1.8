==============================
Chapter 19 - Django Middleware
==============================

Middleware is a framework of hooks into Django's request/response processing.
It's a light, low-level "plugin" system for globally altering Django's input
or output.

Each middleware component is responsible for doing some specific function. For
example, Django includes a middleware component,
:class:`~django.contrib.auth.middleware.AuthenticationMiddleware`, that
associates users with requests using sessions.

This document explains how middleware works, how you activate middleware, and
how to write your own middleware. Django ships with some built-in middleware
you can use right out of the box. See "Available Middleware" later in this
chapter.

Activating middleware
=====================

To activate a middleware component, add it to the
``MIDDLEWARE_CLASSES`` list in your Django settings.

In ``MIDDLEWARE_CLASSES``, each middleware component is represented by
a string: the full Python path to the middleware's class name. For example,
here's the default value created by ``django-admin startproject``::

    MIDDLEWARE_CLASSES = [
        'django.contrib.sessions.middleware.SessionMiddleware',
        'django.middleware.common.CommonMiddleware',
        'django.middleware.csrf.CsrfViewMiddleware',
        'django.contrib.auth.middleware.AuthenticationMiddleware',
        'django.contrib.messages.middleware.MessageMiddleware',
        'django.middleware.clickjacking.XFrameOptionsMiddleware',
    ]

A Django installation doesn't require any middleware —
``MIDDLEWARE_CLASSES`` can be empty, if you'd like — but it's strongly
suggested that you at least use
:class:`~django.middleware.common.CommonMiddleware`.

The order in ``MIDDLEWARE_CLASSES`` matters because a middleware can
depend on other middleware. For instance,
:class:`~django.contrib.auth.middleware.AuthenticationMiddleware` stores the
authenticated user in the session; therefore, it must run after
:class:`~django.contrib.sessions.middleware.SessionMiddleware`. See
middleware-ordering for some common hints about ordering of Django
middleware classes.

Hooks and application order
===========================

During the request phase, before calling the view, Django applies middleware
in the order it's defined in ``MIDDLEWARE_CLASSES``, top-down. Two
hooks are available:

* :meth:`process_request`
* :meth:`process_view`

During the response phase, after calling the view, middleware are applied in
reverse order, from the bottom up. Three hooks are available:

* :meth:`process_exception` (only if the view raised an exception)
* :meth:`process_template_response` (only for template responses)
* :meth:`process_response`

If you prefer, you can also think of it like an onion: each middleware class
is a "layer" that wraps the view.

The behavior of each hook is described below.

Writing your own middleware
===========================

Writing your own middleware is easy. Each middleware component is a single
Python class that defines one or more of the following methods:

.. _request-middleware:

``process_request``
-------------------

.. method:: process_request(request)

``request`` is an :class:`~django.http.HttpRequest` object.

``process_request()`` is called on each request, before Django decides which
view to execute.

It should return either ``None`` or an :class:`~django.http.HttpResponse`
object. If it returns ``None``, Django will continue processing this request,
executing any other ``process_request()`` middleware, then, ``process_view()``
middleware, and finally, the appropriate view. If it returns an
:class:`~django.http.HttpResponse` object, Django won't bother calling any
other request, view or exception middleware, or the appropriate view; it'll
apply response middleware to that :class:`~django.http.HttpResponse`, and
return the result.

.. _view-middleware:

``process_view``
----------------

.. method:: process_view(request, view_func, view_args, view_kwargs)

``request`` is an :class:`~django.http.HttpRequest` object. ``view_func`` is
the Python function that Django is about to use. (It's the actual function
object, not the name of the function as a string.) ``view_args`` is a list of
positional arguments that will be passed to the view, and ``view_kwargs`` is a
dictionary of keyword arguments that will be passed to the view. Neither
``view_args`` nor ``view_kwargs`` include the first view argument
(``request``).

``process_view()`` is called just before Django calls the view.

It should return either ``None`` or an :class:`~django.http.HttpResponse`
object. If it returns ``None``, Django will continue processing this request,
executing any other ``process_view()`` middleware and, then, the appropriate
view. If it returns an :class:`~django.http.HttpResponse` object, Django won't
bother calling any other view or exception middleware, or the appropriate
view; it'll apply response middleware to that
:class:`~django.http.HttpResponse`, and return the result.

.. note::

    Accessing :attr:`request.POST <django.http.HttpRequest.POST>` inside
    middleware from ``process_request`` or ``process_view`` will prevent any
    view running after the middleware from being able to modify the
    upload handlers for the request,
    and should normally be avoided.

    The :class:`~django.middleware.csrf.CsrfViewMiddleware` class can be
    considered an exception, as it provides the
    :func:`~django.views.decorators.csrf.csrf_exempt` and
    :func:`~django.views.decorators.csrf.csrf_protect` decorators which allow
    views to explicitly control at what point the CSRF validation should occur.

.. _template-response-middleware:

``process_template_response``
-----------------------------

.. method:: process_template_response(request, response)

``request`` is an :class:`~django.http.HttpRequest` object. ``response`` is
the :class:`~django.template.response.TemplateResponse` object (or equivalent)
returned by a Django view or by a middleware.

``process_template_response()`` is called just after the view has finished
executing, if the response instance has a ``render()`` method, indicating that
it is a :class:`~django.template.response.TemplateResponse` or equivalent.

It must return a response object that implements a ``render`` method. It could
alter the given ``response`` by changing ``response.template_name`` and
``response.context_data``, or it could create and return a brand-new
:class:`~django.template.response.TemplateResponse` or equivalent.

You don't need to explicitly render responses -- responses will be
automatically rendered once all template response middleware has been
called.

Middleware are run in reverse order during the response phase, which
includes ``process_template_response()``.

.. _response-middleware:

``process_response``
--------------------

.. method:: process_response(request, response)

``request`` is an :class:`~django.http.HttpRequest` object. ``response`` is
the :class:`~django.http.HttpResponse` or
:class:`~django.http.StreamingHttpResponse` object returned by a Django view
or by a middleware.

``process_response()`` is called on all responses before they're returned to
the browser.

It must return an :class:`~django.http.HttpResponse` or
:class:`~django.http.StreamingHttpResponse` object. It could alter the given
``response``, or it could create and return a brand-new
:class:`~django.http.HttpResponse` or
:class:`~django.http.StreamingHttpResponse`.

Unlike the ``process_request()`` and ``process_view()`` methods, the
``process_response()`` method is always called, even if the
``process_request()`` and ``process_view()`` methods of the same middleware
class were skipped (because an earlier middleware method returned an
:class:`~django.http.HttpResponse`). In particular, this means that your
``process_response()`` method cannot rely on setup done in
``process_request()``.

Finally, remember that during the response phase, middleware are applied in
reverse order, from the bottom up. This means classes defined at the end of
``MIDDLEWARE_CLASSES`` will be run first.

Dealing with streaming responses
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Unlike :class:`~django.http.HttpResponse`,
:class:`~django.http.StreamingHttpResponse` does not have a ``content``
attribute. As a result, middleware can no longer assume that all responses
will have a ``content`` attribute. If they need access to the content, they
must test for streaming responses and adjust their behavior accordingly::

    if response.streaming:
        response.streaming_content = wrap_streaming_content(response.streaming_content)
    else:
        response.content = alter_content(response.content)

.. note::

    ``streaming_content`` should be assumed to be too large to hold in memory.
    Response middleware may wrap it in a new generator, but must not consume
    it. Wrapping is typically implemented as follows::

        def wrap_streaming_content(content):
            for chunk in content:
                yield alter_content(chunk)

.. _exception-middleware:

``process_exception``
---------------------

.. method:: process_exception(request, exception)

``request`` is an :class:`~django.http.HttpRequest` object. ``exception`` is an
``Exception`` object raised by the view function.

Django calls ``process_exception()`` when a view raises an exception.
``process_exception()`` should return either ``None`` or an
:class:`~django.http.HttpResponse` object. If it returns an
:class:`~django.http.HttpResponse` object, the template response and response
middleware will be applied, and the resulting response returned to the
browser. Otherwise, default exception handling kicks in.

Again, middleware are run in reverse order during the response phase, which
includes ``process_exception``. If an exception middleware returns a response,
the middleware classes above that middleware will not be called at all.

``__init__``
------------

Most middleware classes won't need an initializer since middleware classes are
essentially placeholders for the ``process_*`` methods. If you do need some
global state you may use ``__init__`` to set up. However, keep in mind a couple
of caveats:

* Django initializes your middleware without any arguments, so you can't
  define ``__init__`` as requiring any arguments.

* Unlike the ``process_*`` methods which get called once per request,
  ``__init__`` gets called only *once*, when the Web server responds to the
  first request.

Marking middleware as unused
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It's sometimes useful to determine at run-time whether a piece of middleware
should be used. In these cases, your middleware's ``__init__`` method may
raise :exc:`django.core.exceptions.MiddlewareNotUsed`. Django will then remove
that piece of middleware from the middleware process and a debug message will
be logged to the ``django.request`` logger when ``DEBUG`` is set to
``True``.

Guidelines
----------

* Middleware classes don't have to subclass anything.

* The middleware class can live anywhere on your Python path. All Django
  cares about is that the ``MIDDLEWARE_CLASSES`` setting includes
  the path to it.

* Feel free to look at Django's available middleware
  for examples.

* If you write a middleware component that you think would be useful to
  other people, contribute to the community! Let us know
  and we'll consider adding it to Django.

Available middleware
====================

Cache middleware
----------------

.. module:: django.middleware.cache
   :synopsis: Middleware for the site-wide cache.

.. class:: UpdateCacheMiddleware

.. class:: FetchFromCacheMiddleware

Enable the site-wide cache. If these are enabled, each Django-powered page will
be cached for as long as the ``CACHE_MIDDLEWARE_SECONDS`` setting
defines. See the cache documentation .

"Common" middleware
-------------------

.. module:: django.middleware.common
   :synopsis: Middleware adding "common" conveniences for perfectionists.

.. class:: CommonMiddleware

Adds a few conveniences for perfectionists:

* Forbids access to user agents in the ``DISALLOWED_USER_AGENTS``
  setting, which should be a list of compiled regular expression objects.

* Performs URL rewriting based on the ``APPEND_SLASH`` and
  ``PREPEND_WWW`` settings.

  If ``APPEND_SLASH`` is ``True`` and the initial URL doesn't end
  with a slash, and it is not found in the URLconf, then a new URL is
  formed by appending a slash at the end. If this new URL is found in the
  URLconf, then Django redirects the request to this new URL. Otherwise,
  the initial URL is processed as usual.

  For example, ``foo.com/bar`` will be redirected to ``foo.com/bar/`` if
  you don't have a valid URL pattern for ``foo.com/bar`` but *do* have a
  valid pattern for ``foo.com/bar/``.

  If ``PREPEND_WWW`` is ``True``, URLs that lack a leading "www."
  will be redirected to the same URL with a leading "www."

  Both of these options are meant to normalize URLs. The philosophy is that
  each URL should exist in one, and only one, place. Technically a URL
  ``foo.com/bar`` is distinct from ``foo.com/bar/`` -- a search-engine
  indexer would treat them as separate URLs -- so it's best practice to
  normalize URLs.

* Handles ETags based on the ``USE_ETAGS`` setting. If
  ``USE_ETAGS`` is set to ``True``, Django will calculate an ETag
  for each request by MD5-hashing the page content, and it'll take care of
  sending ``Not Modified`` responses, if appropriate.

.. attribute:: CommonMiddleware.response_redirect_class

Defaults to :class:`~django.http.HttpResponsePermanentRedirect`. Subclass
``CommonMiddleware`` and override the attribute to customize the redirects
issued by the middleware.

.. class:: BrokenLinkEmailsMiddleware

* Sends broken link notification emails to ``MANAGERS``

GZip middleware
---------------

.. module:: django.middleware.gzip
   :synopsis: Middleware to serve GZipped content for performance.

.. class:: GZipMiddleware

.. warning::

    Security researchers recently revealed that when compression techniques
    (including ``GZipMiddleware``) are used on a website, the site becomes
    exposed to a number of possible attacks. These approaches can be used to
    compromise, among other things, Django's CSRF protection. Before using
    ``GZipMiddleware`` on your site, you should consider very carefully whether
    you are subject to these attacks. If you're in *any* doubt about whether
    you're affected, you should avoid using ``GZipMiddleware``. For more
    details, see the `the BREACH paper (PDF)`_ and `breachattack.com`_.

    .. _the BREACH paper (PDF): http://breachattack.com/resources/BREACH%20-%20SSL,%20gone%20in%2030%20seconds.pdf
    .. _breachattack.com: http://breachattack.com

Compresses content for browsers that understand GZip compression (all modern
browsers).

This middleware should be placed before any other middleware that need to
read or write the response body so that compression happens afterward.

It will NOT compress content if any of the following are true:

* The content body is less than 200 bytes long.

* The response has already set the ``Content-Encoding`` header.

* The request (the browser) hasn't sent an ``Accept-Encoding`` header
  containing ``gzip``.

You can apply GZip compression to individual views using the
:func:`~django.views.decorators.gzip.gzip_page()` decorator.

Conditional GET middleware
--------------------------

.. module:: django.middleware.http
   :synopsis: Middleware handling advanced HTTP features.

.. class:: ConditionalGetMiddleware

Handles conditional GET operations. If the response has a ``ETag`` or
``Last-Modified`` header, and the request has ``If-None-Match`` or
``If-Modified-Since``, the response is replaced by an
:class:`~django.http.HttpResponseNotModified`.

Also sets the ``Date`` and ``Content-Length`` response-headers.

Locale middleware
-----------------

.. module:: django.middleware.locale
   :synopsis: Middleware to enable language selection based on the request.

.. class:: LocaleMiddleware

Enables language selection based on data from the request. It customizes
content for each user. See the internationalization documentation.

.. attribute:: LocaleMiddleware.response_redirect_class

Defaults to :class:`~django.http.HttpResponseRedirect`. Subclass
``LocaleMiddleware`` and override the attribute to customize the redirects
issued by the middleware.

Message middleware
------------------

.. module:: django.contrib.messages.middleware
   :synopsis: Message middleware.

.. class:: MessageMiddleware

Enables cookie- and session-based message support. See the
messages documentation .

.. _security-middleware:

Security middleware
-------------------

.. module:: django.middleware.security
    :synopsis: Security middleware.

.. warning::
    If your deployment situation allows, it's usually a good idea to have your
    front-end Web server perform the functionality provided by the
    ``SecurityMiddleware``. That way, if there are requests that aren't served
    by Django (such as static media or user-uploaded files), they will have
    the same protections as requests to your Django application.

.. class:: SecurityMiddleware

The ``django.middleware.security.SecurityMiddleware`` provides several security
enhancements to the request/response cycle. Each one can be independently
enabled or disabled with a setting.

* ``SECURE_BROWSER_XSS_FILTER``
* ``SECURE_CONTENT_TYPE_NOSNIFF``
* ``SECURE_HSTS_INCLUDE_SUBDOMAINS``
* ``SECURE_HSTS_SECONDS``
* ``SECURE_REDIRECT_EXEMPT``
* ``SECURE_SSL_HOST``
* ``SECURE_SSL_REDIRECT``

.. _http-strict-transport-security:

HTTP Strict Transport Security
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For sites that should only be accessed over HTTPS, you can instruct modern
browsers to refuse to connect to your domain name via an insecure connection
(for a given period of time) by setting the `"Strict-Transport-Security"
header`_. This reduces your exposure to some SSL-stripping man-in-the-middle
(MITM) attacks.

``SecurityMiddleware`` will set this header for you on all HTTPS responses if
you set the ``SECURE_HSTS_SECONDS`` setting to a non-zero integer value.

When enabling HSTS, it's a good idea to first use a small value for testing,
for example, ``SECURE_HSTS_SECONDS = 3600<SECURE_HSTS_SECONDS>`` for one
hour. Each time a Web browser sees the HSTS header from your site, it will
refuse to communicate non-securely (using HTTP) with your domain for the given
period of time. Once you confirm that all assets are served securely on your
site (i.e. HSTS didn't break anything), it's a good idea to increase this value
so that infrequent visitors will be protected (31536000 seconds, i.e. 1 year,
is common).

Additionally, if you set the ``SECURE_HSTS_INCLUDE_SUBDOMAINS`` setting
to ``True``, ``SecurityMiddleware`` will add the ``includeSubDomains`` tag to
the ``Strict-Transport-Security`` header. This is recommended (assuming all
subdomains are served exclusively using HTTPS), otherwise your site may still
be vulnerable via an insecure connection to a subdomain.

.. warning::
    The HSTS policy applies to your entire domain, not just the URL of the
    response that you set the header on. Therefore, you should only use it if
    your entire domain is served via HTTPS only.

    Browsers properly respecting the HSTS header will refuse to allow users to
    bypass warnings and connect to a site with an expired, self-signed, or
    otherwise invalid SSL certificate. If you use HSTS, make sure your
    certificates are in good shape and stay that way!

.. note::
    If you are deployed behind a load-balancer or reverse-proxy server, and the
    ``Strict-Transport-Security`` header is not being added to your responses,
    it may be because Django doesn't realize that it's on a secure connection;
    you may need to set the ``SECURE_PROXY_SSL_HEADER`` setting.

.. _"Strict-Transport-Security" header: http://en.wikipedia.org/wiki/Strict_Transport_Security

.. _x-content-type-options:

``X-Content-Type-Options: nosniff``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some browsers will try to guess the content types of the assets that they
fetch, overriding the ``Content-Type`` header. While this can help display
sites with improperly configured servers, it can also pose a security
risk.

If your site serves user-uploaded files, a malicious user could upload a
specially-crafted file that would be interpreted as HTML or Javascript by
the browser when you expected it to be something harmless.

To learn more about this header and how the browser treats it, you can
read about it on the `IE Security Blog`_.

To prevent the browser from guessing the content type and force it to
always use the type provided in the ``Content-Type`` header, you can pass
the ``X-Content-Type-Options: nosniff`` header.  ``SecurityMiddleware`` will
do this for all responses if the ``SECURE_CONTENT_TYPE_NOSNIFF`` setting
is ``True``.

Note that in most deployment situations where Django isn't involved in serving
user-uploaded files, this setting won't help you. For example, if your
``MEDIA_URL`` is served directly by your front-end Web server (nginx,
Apache, etc.) then you'd want to set this header there. On the other hand, if
you are using Django to do something like require authorization in order to
download files and you cannot set the header using your Web server, this
setting will be useful.

.. _IE Security Blog: http://blogs.msdn.com/b/ie/archive/2008/09/02/ie8-security-part-vi-beta-2-update.aspx

.. _x-xss-protection:

``X-XSS-Protection: 1; mode=block``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some browsers have the ability to block content that appears to be an `XSS
attack`_. They work by looking for Javascript content in the GET or POST
parameters of a page. If the Javascript is replayed in the server's response,
the page is blocked from rendering and an error page is shown instead.

The `X-XSS-Protection header`_ is used to control the operation of the
XSS filter.

To enable the XSS filter in the browser, and force it to always block
suspected XSS attacks, you can pass the ``X-XSS-Protection: 1; mode=block``
header. ``SecurityMiddleware`` will do this for all responses if the
``SECURE_BROWSER_XSS_FILTER`` setting is ``True``.

.. warning::
    The browser XSS filter is a useful defense measure, but must not be
    relied upon exclusively. It cannot detect all XSS attacks and not all
    browsers support the header. Ensure you are still validating and
    all input to prevent XSS attacks.

.. _XSS attack: http://en.wikipedia.org/wiki/Cross-site_scripting
.. _X-XSS-Protection header: http://blogs.msdn.com/b/ie/archive/2008/07/02/ie8-security-part-iv-the-xss-filter.aspx

.. _ssl-redirect:

SSL Redirect
~~~~~~~~~~~~

If your site offers both HTTP and HTTPS connections, most users will end up
with an unsecured connection by default. For best security, you should redirect
all HTTP connections to HTTPS.

If you set the ``SECURE_SSL_REDIRECT`` setting to True,
``SecurityMiddleware`` will permanently (HTTP 301) redirect all HTTP
connections to HTTPS.

.. note::

    For performance reasons, it's preferable to do these redirects outside of
    Django, in a front-end load balancer or reverse-proxy server such as
    `nginx`_. ``SECURE_SSL_REDIRECT`` is intended for the deployment
    situations where this isn't an option.

If the ``SECURE_SSL_HOST`` setting has a value, all redirects will be
sent to that host instead of the originally-requested host.

If there are a few pages on your site that should be available over HTTP, and
not redirected to HTTPS, you can list regular expressions to match those URLs
in the ``SECURE_REDIRECT_EXEMPT`` setting.

.. note::
    If you are deployed behind a load-balancer or reverse-proxy server and
    Django can't seem to tell when a request actually is already secure, you
    may need to set the ``SECURE_PROXY_SSL_HEADER`` setting.

.. _nginx: http://nginx.org

Session middleware
------------------

.. module:: django.contrib.sessions.middleware
   :synopsis: Session middleware.

.. class:: SessionMiddleware

Enables session support. See the session documentation.

Site middleware
---------------

.. module:: django.contrib.sites.middleware
  :synopsis: Site middleware.

.. class:: CurrentSiteMiddleware

Adds the ``site`` attribute representing the current site to every incoming
``HttpRequest`` object. See the sites documentation.

Authentication middleware
-------------------------

.. module:: django.contrib.auth.middleware
  :synopsis: Authentication middleware.

.. class:: AuthenticationMiddleware

Adds the ``user`` attribute, representing the currently-logged-in user, to
every incoming ``HttpRequest`` object. See Authentication in Web requests.

.. class:: RemoteUserMiddleware

Middleware for utilizing Web server provided authentication. See
auth-remote-user for usage details.

.. class:: SessionAuthenticationMiddleware

Allows a user's sessions to be invalidated when their password changes. See
session-invalidation-on-password-change for details. This middleware must
appear after :class:`django.contrib.auth.middleware.AuthenticationMiddleware`
in ``MIDDLEWARE_CLASSES``.

CSRF protection middleware
--------------------------

.. module:: django.middleware.csrf
   :synopsis: Middleware adding protection against Cross Site Request
              Forgeries.

.. class:: CsrfViewMiddleware

Adds protection against Cross Site Request Forgeries by adding hidden form
fields to POST forms and checking requests for the correct value. See the
Cross Site Request Forgery protection documentation .

X-Frame-Options middleware
--------------------------

.. module:: django.middleware.clickjacking
   :synopsis: Clickjacking protection

.. class:: XFrameOptionsMiddleware

Simple clickjacking protection via the X-Frame-Options header .

.. _middleware-ordering:

Middleware ordering
===================

Here are some hints about the ordering of various Django middleware classes:

#. :class:`~django.middleware.cache.UpdateCacheMiddleware`

   Before those that modify the ``Vary`` header (``SessionMiddleware``,
   ``GZipMiddleware``, ``LocaleMiddleware``).

#. :class:`~django.middleware.gzip.GZipMiddleware`

   Before any middleware that may change or use the response body.

   After ``UpdateCacheMiddleware``: Modifies ``Vary`` header.

#. :class:`~django.middleware.http.ConditionalGetMiddleware`

   Before ``CommonMiddleware``: uses its ``Etag`` header when
   ``USE_ETAGS`` = ``True``.

#. :class:`~django.contrib.sessions.middleware.SessionMiddleware`

   After ``UpdateCacheMiddleware``: Modifies ``Vary`` header.

#. :class:`~django.middleware.locale.LocaleMiddleware`

   One of the topmost, after ``SessionMiddleware`` (uses session data) and
   ``CacheMiddleware`` (modifies ``Vary`` header).

#. :class:`~django.middleware.common.CommonMiddleware`

   Before any middleware that may change the response (it calculates ``ETags``).

   After ``GZipMiddleware`` so it won't calculate an ``ETag`` header on gzipped
   contents.

   Close to the top: it redirects when ``APPEND_SLASH`` or
   ``PREPEND_WWW`` are set to ``True``.

#. :class:`~django.middleware.csrf.CsrfViewMiddleware`

   Before any view middleware that assumes that CSRF attacks have been dealt
   with.

#. :class:`~django.contrib.auth.middleware.AuthenticationMiddleware`

   After ``SessionMiddleware``: uses session storage.

#. :class:`~django.contrib.messages.middleware.MessageMiddleware`

   After ``SessionMiddleware``: can use session-based storage.

#. :class:`~django.middleware.cache.FetchFromCacheMiddleware`

   After any middleware that modifies the ``Vary`` header: that header is used
   to pick a value for the cache hash-key.

#. :class:`~django.contrib.flatpages.middleware.FlatpageFallbackMiddleware`

   Should be near the bottom as it's a last-resort type of middleware.

#. :class:`~django.contrib.redirects.middleware.RedirectFallbackMiddleware`

   Should be near the bottom as it's a last-resort type of middleware.

