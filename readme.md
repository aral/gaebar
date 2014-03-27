Please note: I am no longer supporting this project. 

The Google App Engine account that the demo is hosted on is also in the process of being shut down. 

I do not wish to further support a company whose business model is corporate surveillance.

For more information, please see my talk on [digital feudalism](http://thelink.is/avoidingdigitalfeudalism) and
my startup to create a consumer alternative to the closed silos: [Indie Phone](http://indiephone.eu).

Gaebar (Google App Engine Backup and Restore) Beta 3
====================================================

A Naklab™ production sponsored by the <head> web conference - http://headconference.com

Copyright (c) 2009 Aral Balkan. http://aralbalkan.com

Released under the GNU GPL v3 License. See license.txt for the full license or read it here:
http://www.gnu.org/licenses/gpl-3.0-standalone.html


Downloading and Installing Gaebar
=================================

A. From an archive.
-------------------

You can get a .zip or .tar of Gaebar from:
http://github.com/aral/gaebar/tree/master

(Click on the Download link and choose your poison.)

Unzip the Gaebar archive to a folder called gaebar/ off the root of your Django project. You *must* place Gaebar at this location for the app to work properly.


B. From GitHub
--------------

You can install the latest Gaebar trunk into your projects from GitHub using Git.

(a) If you're using Git for your main project
---------------------------------------------

Add Gaebar to your project as a submodule:

git submodule add git://github.com/aral/gaebar-gaed.git 


(b) If you're not using Git for your main project
-------------------------------------------------

Clone Gaebar into a folder called gaebar off the root folder of your project: 

git clone git://github.com/aral/gaebar-gaed.git 

2. To check for updates, go into gaebar/ and git pull

(Don't forget to git commit your main project after you've updated Gaebar to a new version via git pull.)


Configuring your project to use Gaebar
======================================

*IMPORTANT* Patch your dev_appserver.py as per the instructions here: http://aralbalkan.com/1440 (and please star issue 616 if you'd like Google to fix this so we can remove this step: http://code.google.com/p/googleappengine/issues/detail?id=616). 

This is required in order to override some of the local dev server restrictions to allow automatic downloads of backups. Gaebar will not work unless you implement this patch.


1. Add to installed apps
------------------------

Add Gaebar to your list of INSTALLED_APPS in your application's settings.py file. e.g.

	INSTALLED_APPS = (
		# Other apps...
		'gaebar',
	)


2. Add to urls.py
-----------------

In your main urls.py, map the Gaebar app to the URL shown below. You *must* map Gaebar to the exact URL shown below or the app will not work. 

urlpatterns = patterns('',

	# ...other URLs

	url(r'^gaebar/', include('gaebar.urls')),
)

3. Add the static folder
------------------------

In your app.yaml file, add the following entry before any other static entries to map Gaebar's static files (images, js, etc.) correctly:

# Static: Gaebar
- url: /gaebar/static
  static_dir: gaebar/static

4. Add indices
--------------

If you are declaring your indices manually, add the following to your index.yaml file (or run Gaebar locally in the dev server so that the index is created for you automatically):

- kind: GaebarCodeShard
  properties:
  - name: backup
  - name: created_at

(Note: indices may take some time to create on the deployment environment. Until they are ready, backups will fail.)

5. Modify settings.py
---------------------

Modify settings.py to add the GAEBAR_LOCAL_URL, GAEBAR_SECRET_KEY, GAEBAR_SERVERS, and GAEBAR_SERVERS settings to your application.

GAEBAR_LOCAL_URL: Absolute URL of your local development server. Is used when
downloading your remote backup to your local machine.

GAEBAR_SECRET_KEY: A secret key that is used (a) to authenticate communication between your local deployment environment and the remote backup environment to facilitate the download of backups via urlfetch and (b) used during the restore process to authenticate the client.

GAEBAR_SERVERS: Dictionary of named servers. Not essential but makes it easy to identify the servers by name when backing up and restoring. Also makes it easier to identify which server you're running Gaebar on.

GAEBAR_MODELS: Tuple of models from your app that you want backed up.

Here is a sample batch of settings that you can use as a guide:

#
# Gaebar
#

GAEBAR_LOCAL_URL = 'http://localhost:8000'

GAEBAR_SECRET_KEY = 'change_this_to_something_random'

GAEBAR_SERVERS = {
	u'Deployment': u'http://www.myapp.com', 
	u'Staging': u'http://myappstaging.appspot.com', 
	u'Local Test': u'http://localhost:8080',
}

GAEBAR_MODELS = (
     (
          'app1.models', 
          (u'Profile', u'GoogleAccount', u'AllOtherTypes', u'PlasticMan'),
     ),
     (
          'app2.models', 
          (u'Simple',),
     ),
)


A note on templates: 
--------------------

If you get any template-related errors, try adding gaebar/templates to your TEMPLATE_DIRS settings variable.


How it works
============

Gaebar backs up the data in your datastore to Python code. It restores your data by running the generated Python code.

Since a backup is a long running process, and since Google App Engine doesn't support long-running processes, Gaebar fakes a long running process by breaking up the backup and restore processes into bite-sized chunks and repeatedly hitting the server via Ajax calls. 

By default, Gaebar backs up 5 rows at a time to avoid the short term CPU and 10-second call duration quotas and splits the generated code into 
