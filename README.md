# Django App on GCP App Engine Standard with CloudSQL
Sample project to get started with Google Cloud App Engine

I recently deployed a local Django project to the App Engine Standard environment.
This repo shares the steps and sample files explaining how to do this.

## Pre-Conditions:
* Locally working app:  
  Some background about the Django project: 
  * Running in a Python virtual environment 
  * Using SQLite as a DB

* GCP Account:
  * You can use your Gmail account to sign up for GCP.
  * You need a credit card but GCP does not charge you unless you exceed your $300 trial credits. 
  * You can basically click a button saying that you do not want to be charged when the trial credits are gone so you safe.
    There are a few things to get done like billing settings and you will be ready in a few minutes.

## Cloud SDK:
There are multiple ways to interact with GCP. Cloud SDK is one of them and basically a command line interface packaged with other tools and libraries needed to work with GCP. 
Below I will mostly use Cloud SDK for configuring required resources and sometimes will go to the GCP Console, especially for monitoring the status and to see the changes I deployed.
We have to install Cloud SDK to our local machine. I am using Windows 10 and Google has an installer for this.

## Deployment Process: Cloud Infrastructure
* Create a new project:   
  `gcloud projects create my-app-engine-project`
  
* Bind you new project to the billing acount you created:  
`	gcloud alpha billing accounts projects link my-app-engine-project --billing-account=billing-account`  

* Createa an SQL Instnace and a new DB on it:  
	`gcloud --project=my-app-engine-project sql instances create mycloudsql --activation-policy=always --database-version=MYSQL_5_7 --availability-type=zonal --root-password=password --tier=db-f1-micro --zone=europe-west6-a`  
	`gcloud --project=my-app-engine-project sql databases create Django-db --instance=mycloudsql--charset=UTF8  --collation=utf8_general_ci`  
	`gcloud --project=my-app-engine-project sql users create Django-db-user --instance=mycloudsql --password=password`  
	`gcloud --project=my-app-engine-project sql databases list --instance=mycloudsql`  

* Initialize your app:  
  `gcloud app create --project=my-app-engine-project --region=europe-west6`  

## Deployment Process: Database Migration
At this point, we are ready with our infrastructure to host our app on the GCP App Engine.
Before we deploy the app to the GCP, we need to migrate the database using Django manage.py (we have a brand new database in the cloud at this momement).    
   
To make changes on the database in the cloud, we need to create a connection and send local SQL queries to the cloud instead of local SQLite. GCP offers a tool for that: "Cloud SQL Proxy"  
Google and download it to the same directory where your Django project's manage.py is located.  
Open a terminal window in this location and run the following command:  
  
`cloud_sql_proxy_x64.exe -instances"[YOUR_INSTANCE_CONNECTION_NAME]"=tcp:3306`  
  
Replace `YOUR_INSTANCE_CONNECTION_NAME` with the value of `connectionName` in the output of `gcloud --project=my-app-engine-project sql databases list --instance=mycloudsql` command.  
  
Now, we will modify the database settings in the settings.py file of the Django project.  
We do this because we want to use the Cloud SQL if the app is running in the cloud and the SQLite if the app is running locally. Google has already provided a snippet for this. We just modify it for our setup:

```
# [START db_setup]
if os.getenv('GAE_app', None):
    # Running on production App Engine, so connect to Google Cloud SQL using
    # the unix socket at /cloudsql/<your-cloudsql-connection string>
    DATABASES = {
        'default': {
            'ENGINE': 'Django.db.backends.mysql',
            'HOST': '/cloudsql/[YOUR-CONNECTION-NAME]',
            'USER': '[YOUR-USERNAME]',
            'PASSWORD': '[YOUR-PASSWORD]',
            'NAME': '[YOUR-DATABASE]',
        }
    }
    print("Using DB-1(Cloud SQL)")
else:
    # Running locally so connect to either a local MySQL instance or connect 
    # to Cloud SQL via the proxy.  To start the proxy via command line: 
    #    $ cloud_sql_proxy -instances=[INSTANCE_CONNECTION_NAME]=tcp:3306 
    # See https://cloud.google.com/sql/docs/mysql-connect-proxy
    DATABASES = {
        'default': {
            'ENGINE': 'Django.db.backends.sqlite3',
            'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
        }
    }
    print("Using DB-2(Local SQLite)")
# [END db_setup]
```  
  
Above the `connectionName` is the same one as explained in the previous steps. Username/Password pair was also created while we were creating the cloud database above.  
  
"If statement" makes sure that we use the cloud database if we are running in the cloud. In any other case, we assume that we are running the app locally and we use the SQLite.
  
Now, it is time to make some migrations and and apply them to the database. We do this locally using the Django manage.py but Django will make the changes is on the cloud database.  
  
`python manage.py makemigrations`  
  
If Django says there are no new migrations, delete a few past migrations under the my_djangp_app/migrations folder and re-try.
Afterwards, apply the migrations to your Cloud SQL database:  
  
`python manage.py migrate`

You can test the remote databse locally by running the Django server:  
  
`python manage.py runserver`  
  
When you use the web server, this time your data will be fetched from the remote database in the cloud although you are running the app locally. That means we can now deploy our to the cloud.

## Deployment Process: App Deployment
GCP needs to see a few files under your Django project directory to figure out what it needs to do to deploy it in the cloud.  
You Django app should have a folder structure like this:
  
```
./my_project
./my_django_app
./manage.py
```  

We need to create a "app.yaml" and a "main.py" file under the main directory(./).  
  
app.yaml:  
  
```
# [START Django_app]
runtime: python37
handlers:
# This configures Google App Engine to serve the files in the app's
# static directory.
- url: /static
  static_dir: static/
# This handler routes all requests not caught above to the main app. 
# It is required when static routes are defined, but can be omitted 
# (along with the entire handlers section) when there are no static 
# files defined.
- url: /.*
  script: auto
# [END Django_app]
```
  
main.py:
  
```
from my_project.wsgi import app
# App Engine by default looks for a main.py file at the root of the app
# directory with a WSGI-compatible object called app.
# This file imports the WSGI-compatible object of the Django app,
# app from learning_log/wsgi.py and renames it app so it is
# discoverable by App Engine without additional configuration.
# Alternatively, you can add a custom entrypoint field in your app.yaml:
# entrypoint: gunicorn -b :$PORT my_project.wsgi
app = app
```
  
GCP also needs to know what king of dependencies your project needs. We find out them and save in a file in a way that GCP understands as below:  
  
`(v-env)$pip freeze > requirements.txt`  
     
Next, we collect all static app content under a folder with the following command:  
  
`python manage.py collectstatic`  
  
and specify the location of static file so that App Engine knows where to look for them. Add the following line at the bottom of your
/my_project/settings.py file:
  
`STATIC_ROOT = 'static'`  
  
At this point your project folder should include the followings or some more files:  
  
```
./my_project
./my_django_app
./manage.py
./app.yaml
./main.py
./requirements.txt
./static
``` 
We can finally deploy our Django project to the App Engine with a simple gcloud command:  

`gcloud --project=my-app-engine-project app deploy`  

This command will do a few checks and upload your files to the cloud. Subequently it will deploy the app and return a https://<project-id>.<region-id>.r.appspot.com url for your new app running on App Engine.
  
I also uploaded the main.py and app.yaml files I used to this repository.

References: 
* https://cloud.google.com/python/Django/appengine#windows_2
* https://cloud.google.com/appengine/docs/standard/python3/config/appref
* https://github.com/GoogleCloudPlatform/python-docs-samples/tree/master/appengine/standard/Django
* https://github.com/GoogleCloudPlatform/python-docs-samples/tree/master/appengine/flexible/hello_world_Django
* https://medium.com/@BennettGarner/deploying-a-Django-app-to-google-app-engine-f9c91a30bd35
* https://medium.com/@MicroPyramid/django-on-gae-google-app-engine-19e8c5279378
* https://medium.com/@mrdatainsight/performing-database-migrations-with-django-on-google-app-engine-and-cloud-sql-c7fd298581b4
