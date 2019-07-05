# Setting up a Django Project

## Python3
Start by ensuring Python3 is setup.

```
$ python3
Python 3.5.1 (v3.5.1:37a07cee5969, Dec  5 2015, 21:12:44) 
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

All new projects should start with Python 3.x. However, most linux & mac setups 
come pre-installed with Python 2.x. You shouldn't try to update the default python
version as it has lots of OS dependencies. Instead, install Python 3 additionally and 
setup your virtual env to use it for your project.

## Pip - The Python Package Manager
Next up install pip. Pip is used to manage packages and dependencies. Latest instructions
can be found at https://pip.pypa.io/en/stable/installing/

```bash
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python get-pip.py
```

## Virtual Environment
Lets create a virtual environment. Think of virtual environment as an isolated space to
install packages our project would be using. You don't want to do that in system libs as
that makes it difficult to manage versions across projects.

```bash
pip install virtualenv
```

## Create the Project
We would be using an environment variable `$project` for name of the project in rest of the
scripts. Start by setting that up.

```bash
export project = "<Your-Project-Name>"
```

Lets create the project and a virtual environment for it.

```bash
mkdir $project
cd $project
virtualenv -python3 venv
source venv/bin/activate
```

Next, we create a `requirements.txt` file. This will store all package dependencies. Right now, that is just Django.
```
# requirements.txt
Django==2.2.3
```

Now, lets install the dependencies.
```
pip install -r requirements.txt
# python
>>> import django
>>> print(django.get_version())
2.2
```

That will install Django. We will next create a django project using django-admin utility.

```
django-admin startproject <project-name> .
```

Notice the `.` at the end. That exists to avoid a further level of nesting of folders. A couple of more commands and we are ready.

```
python manage.py migrate
python manage.py runserver
# Open http://127.0.0.1:8000/ in browser
```

You can now build and deploy your application in Django, using your IDE of choice, usually [PyCharm](https://www.jetbrains.com/pycharm/download).

And before you push it to a Git repo, add this `.gitignore` file.

```
venv
__pycache__/
*.py[cod]
*.so
*.log
*.pot
db.sqlite3
```


## Production Ready Deployment
The above is just for the development environment. There are a few more steps to get to a production deployment. We will assume container based deployment to AWS. However, since we are building the app in a container, it can be any other cloud provider too.

### Production Settings
Create a file `settings_prod.py` alongside `settings.py` with the following content. These settings are looked up from the deployment environment and should NEVER be checked into the source code.

```py
import os
from settings import *
DEBUG = False

SECRET_KEY = os.environ['SECRET_KEY']
ALLOWED_HOSTS = os.environ['HOST'].split(',')
DATABASES['default']['ENGINE'] = os.environ['DB_ENGINE']
DATABASES['default']['NAME'] = os.environ['DB_NAME']
DATABASES['default']['USER'] = os.environ['DB_USER']
DATABASES['default']['PASSWORD'] = os.environ['DB_PASSWORD']
DATABASES['default']['HOST'] = os.environ['HOST']
DATABASES['default']['PORT'] = os.environ['PORT']
```

### Docker Setup
We will use docker to get this containerised and deployed. Use this Dockerfile. Replace `<project-name>` with the one you used above.

```docker
# Dockerfile
FROM python:3.7.0
ENV PYTHONUNBUFFERED 1
RUN mkdir /code
WORKDIR /code
COPY requirements.txt /code/
RUN pip install --upgrade pip

# Install Deploy Dependencies
RUN pip install psycopg2==2.8.3 
RUN pip install gunicorn==19.9.0
RUN pip install -r requirements.txt

ENV DJANGO_SETTINGS_MODULE <project-name>.settings_prod

COPY . /code/
```

Add the following `.dockerignore` file to avoid copying unnecessary files and folders.

```
.git
venv
__pycache__/
*.py[cod]
*.so
*.log
*.pot
db.sqlite3
```

Build django web image using the dockerfile.

```
docker build -t django_web .
```

Ssh into the docker conatiner to verify if everything is working fine.

```
docker run -it django_web /bin/bash
```

Bookmark the above two commands. You would be using them very often.


### Database Setup
Before running the container, we need to make sure that the database is created, setup and `accessible on the IP address`. The last point is really important if you are trying to run this on a local developer box. Postgres, MySQL and most databases by default are set to listen on localhost IP only. For mysql, the settings are in `mysql.conf`. For Postgres, you would need to edit `postgres.conf` and `pg_hba.conf`.

Do that and verify using a command line client. Once done, lets run the container.

### Running the container
We would be running the container often. So lets create a script with environment variables populated.

```bash
# docker_run.sh
docker run\
 -e SECRET_KEY="<some_long_random_string>"\
 -e HOST="*" \
  -e DB_ENGINE="django.db.backends.postgresql" \
  -e DB_NAME="<DB_NAME>" \
  -e DB_USER="<USER>" \
  -e DB_PASSWORD="<PASSWORD>" \
  -e DB_HOST="database" \
  -e DB_PORT="5432" \
  --add-host=database:<YOUR_DB_IP> \
  -p 8000:8000 \
  django_web \
  "$@"
```

Try that with 
```bash
./docker_run.sh python manage.py runserver
```

If everything worked fine, it should start the server, along with a message
```
You have 17 unapplied migration(s)
```

We had created an empty database. It needs to be initialised. Lets use the container we built to run `migrate` command.

```bash
./docker_run.sh python manage.py migrate
```

# Running Gunicorn
We shouldn't use `runserver` in production. Our dockerfile already had gunicorn installed. Use the below command to run it.

```bash
./docker_run.sh gunicorn -b 0.0.0.0:8000 --workers 4 --name django_web <project-name>.wsgi
```

You should be able to access the application on http://127.0.0.1:8000. It would return a `404 - Not Found` response because we haven't really build any page yet.

# Static Files
That is all fine if all you need is an API server built on Django. However, if you build pages or just want to use admin, we would need to collect and deploy the static files.

TBD

