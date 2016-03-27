---
layout: post
title:  "Tiny MCE in django project."
date:   2016-02-06 10:05:33 -0200
categories: Django
description: "I believe the digital India campaign will promote the country to new heights."
keywords: "Nescode technology, Nescode, Django, TinyMCE, Django framework, Sunil Kumar"
tags: Django
comments: true
---

### Overview
TinyMCE is an undisputed leading platform independent web based WYSIWYG Text editor released under open-source software license. It has the ability to convert HTML textarea field to editor instance where you can play with text styling.

### TinyMCE and Django
Being a Django developer, TinyMCE is my default option to go with because of simplicity and features. One can integrate TinyMCE easily with Django project. I would like to outline the integration in a few easy steps here:

### Step1: Within your virtual environment install the library
{% highlight ruby %}
$ pip install django-tinymce
{% endhighlight %}

### Step2: Add tinymce to INSTALLED_APPS in settings.py for your project:
{% highlight ruby %}
INSTALLED_APPS = (
....
'tinymce',
)
{% endhighlight %}
### Step3: Add tinymce.urls to urls.py for your project
{% highlight ruby %}
urlpatterns = [
 ...
url(r'^tinymce/', include('mcetiny.urls')),
]
{% endhighlight %}

### Step4: In you apps model.py, add required code
{% highlight ruby %}
from django.db import models
from tinymce.models import HTMLField

class MyModel(models.Model):
  offer_descriptions = HTMLField(max_length=300, blank=True, null=True)
{% endhighlight %}

### Step5: Make migrations and migrate to make a change in the database model
{% highlight ruby %}
python manage.py makemigrations
python manage.py migrate
{% endhighlight %}

Thats it! You will have a nice looking text editor with your textarea.

Obviously, you have a whole lot of options available to add many features and formatting options with a TinyMCE text editor. For more details, you may browse publishers github repo (https://github.com/aljosa/django-tinymce) which offers precise guides to implement more in details.
