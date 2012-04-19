This is a simple framework for building fully static, internationalized
websites. If dynamic functionality is needed, a migration path to
[Playdoh][] is available.

## Prerequisites

You need Python version 2.6 or higher. All other dependencies are
self-contained within the project's code repository.

## Setup

Just run this at the terminal prompt:

    cd frodoh
    python manage.py runserver

Then, point your browser to http://localhost:8000/.

## Development

All static, unlocalized files are in the `static` directory, which are
placed at the root of the web site. The `templates` directory
contains localized [Jinja2][] templates that are located at `/<locale>/` on
the web site, where `<locale>` is the name of a locale like `en-US`. The
single exception to this is the file `templates/locale-redirector.html`,
which is used to redirect a non-localized pathname to a localized one (e.g.,
redirecting `/goggles/` to `/en-US/goggles`).

Whenever you need to link to a localized template, you can do so either via
a relative URL or an absolute one that begins with the template variable
`{{ LOCALE_ROOT }}`.

Whenever you need to link to a static file, please use the `{{ STATIC_URL }}` 
template variable. While this is almost always set to the root directory
of your website, it may change, especially if you migrate your site to
Playdoh.

  [Jinja2]: http://jinja.pocoo.org/

## Testing

When writing JavaScript code, please try to make it testable and add
a unit test for it in the `static/test` directory. These [QUnit][]
tests can be run from the development server at  [localhost:8000/test][].

  [QUnit]: http://docs.jquery.com/Qunit
  [localhost:8000/test]: http://localhost:8000/test/

## Localization

The site uses GNU gettext for localization via [Babel][] and Jinja2's
[i18n extension][].

  [Babel]: http://babel.edgewall.org/
  [i18n extension]: http://jinja.pocoo.org/docs/templates/#extensions

## Deployment

Run this at the terminal prompt:

    python manage.py build
    
This will create a static version of the site, for all supported locales, in 
the `dist` directory. You can copy this directory to any web server that 
serves static files, such as Apache or Amazon S3.

## Playdoh Migration

Migrating a site to [Playdoh][] is fairly straightforward due to the
above design philosophy. It requires no dependencies, aside from Playdoh
itself.

You'll want to create a new Django app in your Playdoh project and do
the following:

1. Recursively copy this site's `static` and `templates` directories into the
   Django app.

2. Delete `templates/locale-redirector.html` from the Django app, 
   as Playdoh will now take care of locale redirection for you.

3. Copy any site-specific configuration variables from this site's
   `settings.py` into the Django project's `settings.py`.

4. Fill the Django app's `views.py` with the following helpers:

```python
from django.shortcuts import render
from django.conf import settings
from funfactory.context_processors import i18n

class Locale(object):
    """
    Simple shim class to make locale code in templates work.
    """
    
    def __init__(self, locale, display_name):
        self.locale = locale
        self.display_name = display_name

    def __str__(self):
        return self.locale

def page(filename):
    """
    Simple factory function that returns a Django view for a static website
    page.
    """
    
    def view(request):
        info = i18n(request)
        locales = {}
        for locale, display_name in info['LANGUAGES'].items():
            locales[locale] = Locale(locale, display_name)
        currlocale = locales[info['LANG'].lower()]

        return render(request, filename, {
            'STATIC_URL': settings.STATIC_URL,
            'locales': locales.values(),
            'current_locale': currlocale,
            'LOCALE_ROOT': '/%s/' % currlocale
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

This should expose *only* the site's home page to the Playdoh app. To
expose more pages, you'll need to add to `urlpatterns`.

You *should* be able to just copy the `.po` files from this site into
the Django project and everything will magically work. However, this hasn't
yet been tested.

Note also that you may need to install the [staticfiles][] app in order to get
the above code to work. Alternatively, you should be able to achieve the same
effect by moving the files in `static` into your project's `media` directory
and then setting `STATIC_URL = MEDIA_URL` in your `settings.py`.

  [staticfiles]: https://docs.djangoproject.com/en/dev/ref/contrib/staticfiles/
