---
layout: post
title:  "Setup django project on dreamhost."
date:   2015-07-06 10:05:33 -0200
categories: Django
description: "Django is a web development framework for Python in the same way Rails is a framework for Ruby."
keywords: "Nescode technology, Nescode, Django, Dreamhost, Deploy django on server, Sunil Kumar"
tags: django
comments: true
---

Django is a web development framework for Python in the same way Rails is a framework for Ruby. It is used by a number of company and individual (i.e., for the Google Application Engine), and you can develop rich web applications much easier and faster.

### DreamHost: Brief Intro
DreamHost, A cloud hosting company who offer linux based shared and dedicated virtual machine in relatively cheaper price (Oh Yes!). Just for a trail purpose, I subscribe their shared linux hosting a month back because they offer unlimited PHP, WordPress, Python and Ruby on Rails hosting on one platform including SSH, SFTP, FTP Access. Why I’m impressed with DreamHost is because of their simple approach and workflow.
Start Django deployment

Lets start the Django deployment from beginners perspective.

### Map and Manage domain
Add a domain under Manage domain section, make sure you have checked the Passenger server. Setup your ssh access with dedicated username while mapping domain name

While mapping domain name, DreamHost creates a folder as your domain name. Change directory and move inside the folder.

{% highlight ruby %}

cd domain.com
virtualenv venv # This will create and configure an virtual environment inside your domain directory
source venv/bin/activate # Activate virtualenv
pip install Django==1.8  # You can choose any version
python venv/bin/django-admin.py createproject # This will create a project folder
cd public && mkdir static # Chang directory to public and make a static folder
// Add static folder path in settings.py
STATIC_ROOT = os.path.dirname(BASE_DIR) + '/public/static/'
python manage.py migrate # run migration
touch passenger_wsgi.py # inside domain directory and paste below content with required modification

{% endhighlight %}

```
import sys, os
INTERP = "/home/username/domain.com/venv/local/bin/python"
#INTERP is present twice so that the new python interpreter
if sys.executable != INTERP: os.execl(INTERP, INTERP, *sys.argv)
sys.path.append(os.getcwd() + "/projectname")
os.environ['DJANGO_SETTINGS_MODULE'] = "projectname.settings"
from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```

```
mkdir tmp # Make a folder inside domain directory
cd tmp && touch restart.txt # To restart passenger server
touch tmp/restart.txt # run again to restart passenger server against any changes in passenger_wsgi.py
```
Run above command whenever you will be making any changes in passenger configuration

### Blog post objective

Objective behind writing this blog post is experimenting with DreamHost and their performance. And DreamHost current documentation for Django deployment is outdated. After using almost a month, performance seems to be good. Few other experience which I would prefer to mention as a user:

If you stuck in Django deployment then DreamHost support wont’ give you any help. They will simply redirect you on forum.Even though they have chat support, but whenever you will ping, their average waiting time is more than 7 minute. Support team will respond your email maybe after 12 hours .

### Conclusion
Good to go with DreamHost due to their performance and infrastructure simplicity. Lack of support on configuration is not a big deal I think if you are a developer. Google does a fair job and help you succeed.

### Nescode
Nescode develop web based re-usable application on Django framework. Connect us if you would like to build something awesome.
