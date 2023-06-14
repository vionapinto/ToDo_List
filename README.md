# Deploying to Google Cloud App Engine

## Setup APIs for backing services
App Engine - Main service that runs our application
Secret Manager - to stores secret keys, API Keys, passwords and any other sensitive values
Cloud SQL - Run a PostgreSQL server for all our databases.
Cloud Storage - To store static files (css, js, images)
Cloud Build - compiles our application into App Engine


## initialize App Engine

    ```
    gcloud app create
    ```

## Download SQL Auth Proxy to connect to Cloud SQL from our local machine

1. Authenticate and acquire credentials for the API:

    ```
    gcloud auth application-default login
    ```

2. Download and install the Cloud SQL Auth proxy to our local machine:

    ```
    #Linux 
    curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.3.0/cloud-sql-proxy.linux.amd64
    chmod +x cloud-sql-proxy

    #Windows
    follow this link https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.3.0/cloud-sql-proxy.x64.exe .Rename the file downloaded file to cloud-sql-proxy.exe.
    ```

## Configure environment variables

1. Install django-environ

    ```
    pip install django-environ
    ```

2. Initialize environment using the .env file 

    ```
    import environ

    env = environ.Env(DEBUG=(bool, False))
    env_file = os.path.join(BASE_DIR, '.env')
    env.read_env(env_file)

    SECRET_KEY = env('SECRET_KEY')

    DEBUG = env('DEBUG')

    ALLOWED_HOSTS = ["*"]
    ```

## Configure the Database

1. Install the psycopg database adapter

    ```
    pip install psycopg2-binary
    ```

2. Spin up a PostgreSQL instance in CloudSQL

3. Include the DATABASE_URL variable in the .env file

    DATABASE_URL=postgres://<USERNAME>:<PASSWORD>@//cloudsql/<PROJECT_ID>:<REGION>:<INSTANCE_NAME>/<DATABASE_NAME>

4. Configure the environment variable in settings.py:

    DATABASES = {'default': env.db()}

## Download Gunicorn

```
pip install gunicorn
```

## Create yaml file

```
runtime: python39
env: standard
entrypoint: gunicorn -b :$PORT mysite.wsgi:application

handlers:
- url: /static
  static_dir: static/

- url: /.*
  script: auto

runtime_config:
  python_version: 3
  ```

## Add a .gcloudignore file

```
todo_venv/
__pycache__/
db.sqllite3
.env
.gitignore
```


## Deploy app to App Engine

1. Initialize gcloud CLI

    ```
    gcloud init
    ```

2. Create an App Engine instance

    ```
    gcloud app create
    ```

    follow the prompts to configure to your region


## Connect to our database using Cloud SQL Auth Proxy

    ```
    ./cloud-sql-proxy <PROJECT_ID>:<REGION>:<INSTANCE_NAME>
    ```

## Configure Cloud Storage Bucket

1. Create a Cloud Storage Bucket

2. Add storage bucket name to the .env file

    GS_BUCKET_NAME=<BUCKET_NAME>

## Configure secrets and settings.py to run from Secret Manager

1. Intall cloud secret manager:

    ``` 
    pip install google-cloud-secret-manager
    ```

2. Load environment variables from secret manager into settings.py:


    ```
    # Configurations for .env file
    env = environ.Env(DEBUG=(bool, False))
    env_file = os.path.join(BASE_DIR, '.env')
    # env.read_env(env_file)

    if os.path.isfile(env_file):
        # read the local .env file
        env.read_env(env_file)

    elif os.environ.get('GOOGLE_CLOUD_PROJECT', None):
        # pull the .env from secret manager
        project_id = os.environ.get('GOOGLE_CLOUD_PROJECT')
        
        client = secretmanager.SecretManagerServiceClient()
        settings_name = os.environ.get('SETTINGS_NAME', 'todo_settings')
        name = f'projects/{project_id}/secrets/{settings_name}/versions/latest'
        payload = client.access_secret_version(name=name).payload.data.decode('UTF-8')
        
        env.read_env(io.StringIO(payload))

    else:
        raise Exception('No local .env or GOOGLE_CLOUD_PROJECT detected. No secrets found.')
    ```

3. Create a new secret in secret manager using the .env file

    ```
    gcloud secrets create todo_settings --data-file .env
    ```

4. Configure access to the secret:

    ```
    gcloud secrets add-iam-policy-binding todo_settings --member serviceAccount:python-bugs-1@appspot.gserviceaccount.com --role roles/secretmanager.secretAccessor 

## Configure storage bucket

1. Install django-storages and googl-cloud-storage packages

    ```
    pip install django-storages
    pip install google-cloud-storage
    ```

2. Configure static files in settings.py

    ```
    GS_BUCKET_NAME = env('GS_BUCKET_NAME')
    STATIC_URL = "static/"
    DEFAULT_FILE_STORAGE = "storages.backends.gcloud.GoogleCloudStorage"
    STATICFILES_STORAGE = "storages.backends.gcloud.GoogleCloudStorage"
    GS_DEFAULT_ACL = "publicRead"
    STATICFILES_DIRS = [BASE_DIR/'static/']
    STATIC_ROOT = BASE_DIR/'staticfiles'
    ```

3. Collect static files into storage bucket

    ```
    python manage.py makemigrations
    python manage.py migrate
    python manage.py collectstatic
    ```

## Deploy to App Engine

```
gcloud app deploy
```

```
gcloud app browse
```



