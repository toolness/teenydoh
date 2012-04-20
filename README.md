Teenydoh is a simple tool for building fully static, internationalized
websites. If dynamic server-side functionality is needed, a migration path to
[Playdoh][] is available.

  [Playdoh]: https://github.com/mozilla/playdoh

## Use Cases

Teenydoh is currently the foundation for the following websites:

  * [hackasaurus.org](http://hackasaurus.org/)
    ([github](https://github.com/hackasaurus/hackasaurus.org))
  * [openbadges.org](http://openbadges.org/)
    ([github](https://github.com/mozilla/openbadges.org))

## Prerequisites

You just need Python version 2.6 or higher. All other dependencies are
self-contained within the project's code repository.

## Setup

Just run this at the terminal prompt:

    git clone git://github.com/toolness/teenydoh.git
    cd teenydoh
    python manage.py runserver

Then, point your browser to http://localhost:8000/.

## Development

All static, unlocalized files are in the `static` directory.

The `templates` directory contains localized [Jinja2][] templates that are
located at `/<locale>/` on your web site, where `<locale>` is the name of a 
locale like `en-US`. The single exception to this is the locale redirector
template, explained later in this document.

The following template variables are defined:

* `{{ LOCALE_ROOT }}` is the locale-specific root of your site. Whenever you 
  need to link to a localized template, you can do so either via a relative
  URL or an absolute one that begins with this template variable. An example
  value for this variable is `/en-US/`.

* `{{ STATIC_URL }}` is the root directory for the files in your site's
  `static` directory. Example values of this are `/` and `/static/`.

* `{{ LANGUAGES }}` is a dictionary of all available languages, such as 
  `{'en-us': 'English (US)'}`.

* `{{ LANG }}` is the currently active language code, such as `en-US`.

* `{{ DIR }}` is `ltr` if the active language reads left-to-right. Otherwise, 
  it is `rtl`.

* `{{ settings }}` contains all the variables, if any, defined in your
  site's `settings.py`.

### The Locale Redirector Template

The locale redirector template at `templates/locale-redirector.html` is
used to redirect a non-localized pathname to a localized one (e.g., 
redirecting `/goggles/` to `/en-US/goggles/`). It's automatically generated
for any path containing a localized template called `index.html`.

The path that the redirector must redirect to is in the
`{{ PATH_INFO }}` template variable. This template doesn't have `{{ LANG }}`
or `{{ DIR }}` defined, because the actual locale isn't known at the time
that this template is accessed: since the generated site is entirely static
content capable of being served from any stock server, content negotiation
must occur on the client-side using JavaScript.

  [Jinja2]: http://jinja.pocoo.org/

## Testing

When writing JavaScript code, you can add unit tests in the `static/test` 
directory. These [QUnit][] tests can be run from the development server
at [localhost:8000/test][].

  [QUnit]: http://docs.jquery.com/Qunit
  [localhost:8000/test]: http://localhost:8000/test/

## Localization

Teenydoh uses GNU gettext for localization via [Babel][] and Jinja2's
[i18n extension][].

To create a new locale for e.g. Hebrew (`he`), run:

    python manage.py makemessages -l he

Then, edit `locale/he/LC_MESSAGES/messages.po` as needed. When you're done
localizing, run:

    python manage.py compilemessages

This will convert all available `.po` files into `.mo` files. The next
time you run the development server or rebuild the static site, your new
translations will be used.

You can also use a site like [localize.mozilla.org][] to make it easy
for community members to contribute localizations.

  [Babel]: http://babel.edgewall.org/
  [i18n extension]: http://jinja.pocoo.org/docs/templates/#extensions
  [localize.mozilla.org]: http://localize.mozilla.org/

## Deployment

Run this at the terminal prompt:

    python manage.py build
    
This will create a static version of your site, for all supported locales, in 
the `dist` directory. You can copy this directory to any web server that 
serves static files, such as Apache or Amazon S3.

## Playdoh Migration

Migrating a site to [Playdoh][] is fairly straightforward. It requires no dependencies, aside from Playdoh itself.

You'll want to create a new Django app in your Playdoh project and do
the following:

1. Recursively copy your site's `static` and `templates` directories into the
   Django app.

2. Delete `templates/locale-redirector.html` from the Django app, 
   as Playdoh will now take care of locale redirection for you.

3. Copy any site-specific configuration variables from your site's
   `settings.py` into the Django project's `settings.py`.

4. Fill the Django app's `views.py` with the following:

```python
from django.shortcuts import render
from django.conf import settings
from funfactory.context_processors import i18n

def page(filename):
    """
    Simple factory function that returns a Django view for a static website
    page.
    """
    
    def view(request):
        info = i18n(request)
        return render(request, filename, {
            'STATIC_URL': settings.STATIC_URL,
            'LOCALE_ROOT': '/%s/' % info['LANG']
        })
    
    return view
```

5. Fill the Django app's `urls.py` with the following:

```python
    from django.conf.urls.defaults import *

    from .views import page

    urlpatterns = patterns('',
        url(r'^$', page('index.html')),
    )
```

This should expose *only* your site's home page to the Playdoh app. To
expose more pages, you'll need to add to `urlpatterns`.

You *should* be able to just copy the `.po` files from your site into
the Django project and everything will magically work. However, this hasn't
yet been tested.

Note also that you may need to install the [staticfiles][] app in order to get
the above code to work. Alternatively, you should be able to achieve the same
effect by moving the files in `static` into your project's `media` directory
and then setting `STATIC_URL = MEDIA_URL` in your `settings.py`.

  [staticfiles]: https://docs.djangoproject.com/en/dev/ref/contrib/staticfiles/
