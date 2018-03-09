---
layout: post
title:  "Sharing Jinja templates between Python / Flask applications"
date:   2018-03-09 11:15:41 +0000
categories: python flask jinja govuk
---
I have been doing some work recently on our frontend applications which involved creating some [Jinja](http://jinja.pocoo.org/docs/2.10/) macros enable us to quickly and easily write GOV.UK compliant form markup. For example this is how to use the macro to create a simple password field (where form is a [Flask-WTF](https://flask-wtf.readthedocs.io/en/stable/) form):

{% highlight jinja %}
{% raw %}
{{ element(
    form,
    name='password',
    attributes={'type': 'password'}
) }}
{% endraw %}
{% endhighlight %}

Having the code for the underlying macro in each application seems a little unnecessary and so in the spirit of [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself), I investigated the following method for pulling this code out into it's own repository. You can find this repo here if you want to skip straight to it:

[https://github.com/LandRegistry/govuk-elements-jinja-macros](govuk-elements-jinja-macros)

Otherwise, here's the breakdown of what's going on:

## Setting up a python package to contain the templates

The first step is to take the templates we want to share and put them into a Python package. I'm not going to go into all the details here, others have already written more eloquently on the subject (Google things like `python package boilerplate` or `creating python package`. I will however highlight the important bits for our use case.

Assuming a setup.py that contains something like this:

{% highlight python %}
setuptools.setup(name='govuk-elements-jinja-macros',
                 version='1.0.7',
                 description='GOV.UK elements Jinja macros',
                 packages=['govuk_elements_jinja_macros'],
                 package_data={'govuk_elements_jinja_macros': ['templates/*.html']}
                 )
{% endhighlight %}

The pertinent line here that goes beyond the "simplest" package structure is the `package_data` line which is a dict which maps package names to glob patterns. In this case, we want to make sure that when you `pip install` this package, you get a copy of all the html files. There's actually no Python in this Python package!

_There are [other ways of including package data](http://setuptools.readthedocs.io/en/latest/setuptools.html#including-data-files) such as a blanket `include_package_data=True`, this just happens to be the method I chose as it allows more fine grained control._

You should then put your templates inside the `govuk_elements_jinja_macros/templates` folder. We will expose these to our Flask app in the next step.

## Installing the templates

Once you've set up your template package and pushed it to a git repository somewhere, you will need to add it to your `requirements.txt`. Unless you're going all the way and pushing it to PyPI, you can just pop a URL to the package archive in your requirements like this:

```
https://github.com/LandRegistry/govuk-elements-jinja-macros/archive/v1.0.7.zip
```

_There are a variety of ways to do this, including via git/ssh - [refer to the pip documentation for other options](https://pip.pypa.io/en/stable/reference/pip_install/#requirements-file-format). Worth noting that if you are using [pip-compile](https://github.com/jazzband/pip-tools), it doesn't play well with these non standard formats so you may wish to use the `CUSTOM_COMPILE_COMMAND` documented at https://github.com/jazzband/pip-tools#configuration in order to manage additional dependencies like this_

Registering the installed templates into your app requires a slight restructure to the default Flask setup. Flask defaults to instructing Jinja to look in it's standard location of `/templates` to load your templates, but we need to override this. In order for our installed templates to live happily alongside our application's templates we need to register them on a prefix as follows:

{% highlight python %}
from jinja2 import PackageLoader, PrefixLoader

app.jinja_loader = PrefixLoader({
    'app': PackageLoader('name_of_app_folder'),
    'govuk_elements_jinja_macros': PackageLoader('govuk_elements_jinja_macros')
})
{% endhighlight %}

The `'app': PackageLoader('name_of_app_folder')` line will look for templates in the `name_of_app_folder/templates` directory. If your structure doesn't match this, you can pass a second argument to `PackageLoader` instructing it to look in a different sub folder of your package such as `PackageLoader('name_of_app_folder', 'my/templates/live/here')`.

Once this is set up, where you would normally write `{% raw %}{% include foo.html %}{% endraw %}` in your templates, you now need to access this template with the prefix like `{% raw %}{% include app/foo.html %}{% endraw %}`. This applies to render_template calls as well where you will need to say `render_template('app/foo.html')`.

Similarly the `'govuk_elements_jinja_macros': PackageLoader('govuk_elements_jinja_macros')` will register the templates from the package we installed earlier onto the `govuk_elements_jinja_macros` prefix. So these will then become available to use like `{% raw %}{% from 'govuk_elements_jinja_macros/form.html' import element %}{% endraw %}`

## Other use cases

Other good uses cases include design systems / pattern libraries. As well as defining CSS & JS, your pattern library could expose templates for your application to use. The benefit of this is that your markup and associated assets are kept in step so changes to structure upstream would automatically get rolled out to your application.