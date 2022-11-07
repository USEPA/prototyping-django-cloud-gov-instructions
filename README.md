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
5. Edit models.py
6. Add to urls.py:
```
from .routers import router
from rest_framework.schemas import get_schema_view
```
   - And within urlpatterns:
```
    path('api/', include(router.urls)),
    path('openapi', get_schema_view(
            title="crosswalks",
            description="REST APIs",
            version="1.0.0"
    ), name='openapi-schema'),
```
7. Create serializers.py within ***appName*** and add a modelClassName for ever class in your models.py:
```
from rest_framework import serializers
from .models import ***ModelClassName***

class ***ModelClassName***Serializer(serializers.ModelSerializer):
    class Meta:
        model = ***ModelClassName***
        fields = '__all__'
```
8. Add a viewsets.py within ***appName***:
```
from rest_framework import viewsets
from .models import ***ModelClassName***
from .serializers import ***ModelClassName***Serializer

class ***ModelClassName***ViewSet(viewsets.ModelViewSet):
    queryset = ***ModelClassName***.objects.all()
    serializer_class = ***ModelClassName***Serializer
```
9. Create a routers.py within ***projectName/projectName*** (should be collocated with urls.py):
```
from rest_framework import routers
from ***appName***.viewsets import ***ModelClassName***ViewSet

router = routers.DefaultRouter()
		
router.register(r'***ModelClassName***', ***ModelClassName***ViewSet)
```
10. Create a manifest.yml file within your top-level project folder:
```
---
applications:
- name: ***projectName***
  memory: 256M
  instances: 1
  command: bash ./run.sh
  buildpacks: 
  - python_buildpack
  services:
  - ***databaseServiceName***
  - ***credentialsServiceName***
```
11. Create a run.sh file within the top-level project directory:
```
#!/bin/bash
echo "make migrations"

python manage.py makemigrations

echo "migrate"

python manage.py migrate

# If admin account exists, reset its password according to credentials provided, otherwise create it
cat <<EOF | python manage.py shell
import os, json
from django.contrib.auth.models import User

creds = json.loads(os.getenv('VCAP_SERVICES'))['user-provided'][0]['credentials']

u = User.objects.filter(username=creds['username'])

if u.exists():
    u[0].set_password(creds['password'])
else:
    User.objects.create_superuser(creds['username'], creds['email'], creds['password'])
EOF

echo "Starting Django Server..."

python manage.py runserver 0.0.0.0:$PORT
```
12. Create a Procfile within the top-level project directory:
```
web: gunicorn ***projectName***.wsgi
```
13. Create a requirements.txt within the top-level project directory:
```
djangorestframework
psycopg2
gunicorn
pyyaml 
uritemplate
django-cors-headers
Whitenoise
```
14. Create a database service (name must match your manifest.yml) in the command line:
```
cf create-service aws-rds micro-psql ***databaseServiceName***
```
15. Create a credentials service within the command line:
```
cf cups ***credentialsServiceName*** -p "{\"username\":\"***adminUsername***\",\"email\":\"noreply@epa.gov\",\"password\":\"***adminPassword***\",\"secret_key\":\"***secretKey***\"}"
```
16. Collect static files:
```
python manage.py collectstatic
```
17. Run migrations locally and check that the site is working:
```
python manage.py makemigrations
python manage.py migrate
python manage.py runserver
```
18. Push to cloud.gov sandbox:
```
cf push -f manifest.yml
```
