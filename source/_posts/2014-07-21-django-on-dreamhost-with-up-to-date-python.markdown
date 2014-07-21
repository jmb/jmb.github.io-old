---
layout: post
title: "Django on Dreamhost with up-to-date python"
date: 2014-07-21 15:37:51 +0100
comments: true
categories: [Django, Python, Dreamhost]
---
I recently went through the process of installing my own python (2.7.7) and Django (1.6.5) versions on my Dreamhost account in order to be on the same versions as I'm running in development.

I used [this guide from stackexchange][1] as a basis, with a few tweaks.

Building a custom python install
--------------------------------

SSH into your shell account

	cd ~
	mkdir tmp
	cd tmp
	wget http://www.python.org/ftp/python/2.7.7/Python-2.7.7.tgz
	tar zxvf Python-2.7.7.tgz
	cd Python-2.7.7
	./configure --prefix=$HOME/opt/python-2.7.7
	make
	make install
This will install your local version of python to ```/home/<username>/opt/python-2.7.7```

Add to your path in order to use this version of python over the system default: ```export PATH=$HOME/opt/python-2.7.7/bin:$PATH```. You can also add this line to the .bashrc and/or .bash_profile file in your home directory to make this permanent. ([ see here for info on the difference][2]) Other shells are available, but I'll assume if you've changed yours you really know what you're doing!

Check which version of python you're now using: ```python -V```

In order to install various python packages, and Django in particular, then you'll want [pip][3].

	curl https://bootstrap.pypa.io/get-pip.py > ~/tmp/get-pip.py
	python ~/tmp/get-pip.py
(Using curl here rather than wget, as it handles the secure site (https) better)

Finally in order to "freeze" the package versions for your website (so they don't automatically update when you set up another site at a later date), you'll likely want to install virtualenv: ```pip install virtualenv```

Setting up your Django site
---------------------------

In the [Dreamhost Panel][4] set up a new fully hosted domain with a web directory set to /home/username/```<domain.name>/public``` and enable Passenger.
Once the new directory has been created on your server, set up a new virtualenv:
	
	virtualenv $HOME/<domain.name>/env
This will create a local copy of your environment specific to this website.

While working on this website, you should activate the local environment in order to make sure you're working with the right versions of your tools and packages:

	source $HOME/<domain.name>/env/bin/activate
Now you can install Django and any other required packages (e.g. MySQL-python if you're going to use a MySQL database) for your website using pip:

	pip install Django MySQL-python
Now you can create your Django project:

	cd $HOME/<domain.name>/
	python env/bin/django-admin.py startproject <projectname>
In order for Passenger to pick up your project, you need to create a ```passenger_wsgi.py``` file within your site's top level directory (```/home/<username>/<domain.name>```).
The file should contain the following:

	import sys, os
	cwd = os.getcwd()
	sys.path.append(cwd)
	sys.path.append(os.path.join(cwd, '/<projectname>'))  #You must add your project here
 
	#Switch to new python
	if sys.version < "2.7.7": os.execl(os.path.join(cwd,"/env/bin/python"), "python2.7", *sys.argv)

	sys.path.insert(0, os.path.join(cwd'/env/bin'))
	sys.path.insert(0, os.path.join(cwd'/env/lib/python2.7/site-packages/django'))
	sys.path.insert(0, os.path.join(cwd'/env/lib/python2.7/site-packages'))

	os.environ['DJANGO_SETTINGS_MODULE'] = "<projectname>.settings"
	import django.core.handlers.wsgi
	application = django.core.handlers.wsgi.WSGIHandler()
	
Be sure to replace the two instances of ```<projectname>``` with your actual project name!


You should set up django's static files settings in order to correctly serve images, css and javascript - you will need this for the admin interface to work for example.

Within the projects setting file, found at ```<projectname>/<projectname>/settings.py``` you will find that the ```STATIC_URL``` is probably configured to ```/static/```.
Add another line to set the location on the server of the actual static directory:

	STATIC_ROOT = os.path.join(os.path.dirname(BASE_DIR), '/public/static/')
This will be the location where Django will put all of your static files - you shouldn't put stuff here manually as it gets overwritten. Run the collectstatic command now to set up the static items for the admin interface:

	cd $HOME/<domain>/
	<projectname>/manage.py collectstatic
Finally you should set up your database as required within the settings.py file - the default is to use sqlite3, which may be suitable for the smallest of sites, but it's likely you'll want to set up a mysql database.
Once configured, run a syncdb:

	<projectname>/manage.py syncdb
Set up a superuser as and if required.

Et voila! You should now see the standard Django holding page at your domain and be able to access the admin console at ```http://domain.name/admin/```.

[1]: http://stackoverflow.com/questions/10953695/update-new-django-and-python-2-7-with-virtualenv-on-dreamhost-with-passenger
[2]: http://www.joshstaiger.org/archives/2005/07/bash_profile_vs.html
[3]: https://pip.pypa.io/en/latest/installing.html
[4]: https://panel.dreamhost.com