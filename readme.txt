Gaebar (Google App Engine Backup and Restore) Beta 1
====================================================

A Naklab™ production sponsored by the <head> web conference - http://headconference.com

Copyright (c) 2009 Aral Balkan. http://aralbalkan.com

Released under the GNU GPL v3 License. See license.txt for the full license or read it here:
http://www.gnu.org/licenses/gpl-3.0-standalone.html



Limitations
===========

Ancestor relationships are not supported. Quite frankly, I haven't gotten my head around the whole ancestor thing at all. I'm going to look into it again today to see if I can't implement it.

Feedback and patches are always welcome! :)



Installation
============

1. *IMPORTANT* Patch your dev_appserver.py as per the instructions here: http://aralbalkan.com/1440 (and please star issue 616 if you'd like Google to fix this so we can remove this step: http://code.google.com/p/googleappengine/issues/detail?id=616). 

This is required in order to override some of the local dev server restrictions to allow automatic downloads of backups. Gaebar will not work unless you implement this patch.


2. Unzip Gaebar.zip to a folder called gaebar/ off the root of your Django project. You *must* place Gaebar at this location for the app to work properly.


3. Add Gaebar to your list of INSTALLED_APPS in your application's settings.py file. e.g.

	INSTALLED_APPS = (
		# Other apps...
		'gaebar',
	)


4. In your main urls.py, map the Gaebar app to the URL shown below. You *must* map Gaebar to the exact URL shown below or the app will not work. 

urlpatterns = patterns('',

	# ...other URLs

	url(r'^gaebar/', include('gaebar.urls')),
)

5. In your app.yaml file, add the following entry to map Gaebar's static files (images, js, etc.) correctly:

# Static: Gaebar
- url: /gaebar/static
  static_dir: gaebar/static



Settings
========

Before you start using Gaebar, you have to configure it by added a few lines to your Django application's settings file (settings.py). The settings, with sample values, are shown below:

#
# Gaebar
#

GAEBAR_LOCAL_URL = 'http://localhost:8000'

GAEBAR_BACKUPS_FOLDER = '/Users/aral/projects/gaebar-gaed/gaebar/backups/'

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



Testing Gaebar locally:
=======================

(Optional) If you want to test Gaebar locally, you need to run two instances of the local development server. You will use one as your local development server and the other you will pretend is the deployment server that you want to back up.

*Important* I had to implement the following hack in order to get local tests to run correctly while using appenginepatch: http://aralbalkan.com/1755. You may want to keep it in mind in case you run into the same issue.

I usually run a server on port 8000 to act as the local development server and port 8080 to act as the remote server. 

Add the following into your settings file, substituting the ports of your servers:

GAEBAR_LOCAL_URL = 'http://localhost:8000'

GAEBAR_SERVERS = {
	u'Local Test': u'http://localhost:8080',
	# Other servers...
}

Populate your datastore with some sample data then hit your faux remote server (e.g., http://localhost:8080/gaebar/) and start a backup. It should back up the datastore and then hit the local server (port 8000) to download the backup. You will find your backup files in the gaebar/backups folder (as specified by settings.GAEBAR_BACKUPS_FOLDER).



How it works
============

Gaebar backs up the data in your datastore to Python code. It restores your data by running the generated Python code.

Since a backup is a long running process, and since Google App Engine doesn't support long-running processes, Gaebar fakes a long running process by breaking up the backup and restore processes into bite-sized chunks and repeatedly hitting the server via Ajax calls. 

By default, Gaebar backs up 5 rows at a time to avoid the short term CPU and 10-second call duration quotas and splits the generated code into code shards of approx. 300KB to avoid the 1MB limit on objects. You can change these defaults in the views.py file if your app has higher quotas and you want faster backups and restores. 

Gaebar only works with Django applications on Google App Engine. Both appenginepatch and App Engine Helper are supported.

Please test Gaebar out with sample data locally before testing it on your live app. We cannot be held responsible for any data loss or other damage that may arise from your use of Gaebar.



Usage
=====

A. To make a remote backup:
---------------------------

1. Deploy Gaebar, along with your application, to Google App Engine.

NOTE: When you're deploying, remember that you will also deploy any backups that are in the gaebar/backups folder. It's a good idea to move these elsewhere before deploying to create a new backup to reduce the number of files in your app. and only deploy with the backup you want to restore to reduce the number of files in your app so as not to hit the 1,000 file limit. You cannot harm the app by moving or deleting backup folders.

2. Hit the Gaebar index page from the Google App Engine deployment environment (e.g., http://myapp.appspot.com/gaebar/)

3. Click the Create New Backup button and wait.*

* Make sure that your local dev server is running so that your backup can be downloaded to your local machine.

Note: If your datastore has reference errors, i.e., non-existent references which would result in a 'ReferenceProperty failed to be resolved' error, the backup should fix those errors by ignoring those reference properties that do not exist. When you restore a Gaebar backup, those reference errors should no longer exist. You will see comments in the code shards when reference errors are encountered. 

Note: The largest datastore I've tested this with is for the <head> conference web site. The latest backup contained 18,955 rows stored in 223 code shards and resulted in a 35MB .datastore file when restored on the local development server (the restore process was left to run overnight due to the speed of the local datastore). Please send statistics of your backups and, of course, any errors you may encounter to aral@aralbalkan.com.


B. Restore data locally:
------------------------

You may want to restore data locally, on your development server, to test with real-world data while developing your application. To do this, 

1. Hit the Gaebar index page on your local machine (e.g., http://localhost:8000/gaebar/)

2. Select a backup to restore from the list.

3. Wait until the backup is restored.

*** Tip: *** The datastore and history are kept in temporary folders such as (on OS X) /var/folders/bz/bzDU030xHXK-jKYLMXnTzk+++TI/-Tmp-/. Before you do a restore, run ./manage.py flush to find out which folder they're kept in. Once you have that, copy those files to a safe place. This way, you can easily restore them if they are erased without having to go through the lengthy restore process again.

Also, realize that the datastore on the local SDK is _very_ slow. Launching the server with a large datastore will take a long time. Even doing a ./manage.py flush will take ages, and you can get the same result by simply deleting the datastore files from the temporary folder.


C. Restore data to a staging app:
---------------------------------

Although Google App Engine gives you basic versioning control during deployment, it doesn't provide a staging environment where you can test out updates with real data on the deployment server without your end users seeing.

With Gaebar, you can set up your own staging application. Simply set up a separate application, clone your app folder (changing the app name), and deploy. Then, backup your data from your main app. And deploy again to upload your backups to the staging app. On the staging app, restore the data and you can test your latest changes with real data before exposing those changes to your users.

NOTE: When you're deploying, remember that you will also deploy any backups that are in the gaebar/backups folder. It's a good idea to only deploy with the backup you want to restore to reduce the number of files in your app so as not to hit the 1,000 file limit. You cannot harm the app by moving or deleting backup folders.


D. Restore data to your main app:
--------------------------------

If something happens to your main application, you can restore from a backup.

*** Please note that this will REPLACE the data in your main datastore *** THIS IS A DESTRUCTIVE OPERATION. Make sure you fully understand the ramifications before restoring on your deployed application. We take no responsibility for data loss or other damage caused by your use of this software.

NOTE: When you're deploying, remember that you will also deploy any backups that are in the gaebar/backups folder. It's a good idea to only deploy with the backup you want to restore to reduce the number of files in your app so as not to hit the 1,000 file limit. You cannot harm the app by moving or deleting backup folders.



Known Issues
============

* Ancestor relationships are untested/unsupported.

