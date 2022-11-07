# Instructions for a simple Django project in cloud.gov for prototyping APIs
These directions are used to create a very minimal (and not production-ready) set of APIs in a cloud.gov sandbox to help refine data models.
It uses Django Rest Framework for the APIs and the Djano admin site to allow users to see the data model. It sets up a single admin account, so the security is limited.
## Final project set-up
Note ***bold, italic*** (or three *s within code) text indicates you should replace it with your projectName/appName.

***projectName***:
- manage.py
- manifest.yml
- Procfile
- requirements.txt
- run.sh
- .cfignore
- .gitignore
- ***projectName***:
  - asgi.py
  - routers.py
  - settings.py
  - urls.py
  - wsgi.py
 - ***appName***:
   - admin.py
   - apps.py
   - models.py
   - serializers.py
   - tests.py
   - views.py
   - viewsets.py

## Steps
1. Start the project in the python command line
```
django-admin startproject ***projectName***
cd ***projectName***
python manage.py startapp ***appName***
```
2. Add your .cfignore file to block from pushing to cloud.gov
```
db.sqlite3
```
3. Copy .cfignore to .gitignore
4. Edit settings.py
   - Add:
```
import os, json
```
   - Add to INSTALLED_APPS:
```
    'whitenoise.runserver_nostatic',
    '***appName***',
    'rest_framework',
    'corsheaders',
```
   - Within MIDDLEWARE (just after SecurityMiddleware), add:
```
    'whitenoise.middleware.WhiteNoiseMiddleware',
    'corsheaders.middleware.CorsMiddleware',
```
   - Replace DATABASES with:
```
if os.getenv('VCAP_SERVICES'):
    print("Running on cloud.gov!")
    DEBUG = False
    creds = json.loads(os.getenv('VCAP_SERVICES'))['user-provided'][0]['credentials']
    SECRET_KEY = creds['secret_key']
    uri = 'https://' + json.loads(os.getenv('VCAP_APPLICATION'))['application_uris'][0]
    CSRF_TRUSTED_ORIGINS=[uri]
else:
    print("Running locally")
    DEBUG = True
    try:
        f = open("../local_key.txt", "r")
        SECRET_KEY = f.read().strip()
    except:
        f = open("../local_key.txt", "w")
        from django.core.management.utils import get_random_secret_key
        SECRET_KEY = get_random_secret_key()
        f.write(SECRET_KEY)
    f.close()
```
   - Require authentication for the APIs by adding:
```
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    ),
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.BasicAuthentication',
    ),
}
```
   - Allow CORS for all origins by adding:
```
CORS_ALLOW_ALL_ORIGINS = True
```
   - Update ALLOWED_HOSTS to:
```
ALLOWED_HOSTS = ['*']
```
   - Above STATIC_URL add:
```
STATIC_ROOT = BASE_DIR / "staticfiles"
```
