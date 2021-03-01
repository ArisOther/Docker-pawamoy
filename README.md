# DOCKER
## Architecture plan

    ```
                                
        ├── [nginx]
        ├── --bridge1
        ├── [django + gunicorn]
                            |                
                            ├── bridge1 -->[postgres1]-->volume_db1            
                            └── bridge2 -->[postgres2]-->volume_db2
                
    ```
## Instalasi django dan gunicorn
- Instalasi Django
    - django-admin startproject hello
    - cd hello
    - python manage.py runserver
    - tes http://localhost:8000
- instalasi gunicorn local
    - pip install gunicorn
    - gunicorn --bind :8000 hello.wsgi:application
        - note: gunicorn menggantikan fungsi runserver dalam merender WSGI
## Docker image - hello app
- Buat Dockerfile
- Dockerfile tree
    ```
                            # Your current directory, created for this tutorial
    ├── hello                 # The Django project
    │   ├── hello             # The main Django app of your project
    │   └── manage.py
    └── Dockerfile            # Your Dockerfile
    ```
- Dockerfile
    ```docker
    # start from an official image
    FROM python:3.8

    # arbitrary location choice: you can change the directory
    RUN mkdir -p /opt/services/djangoapp/src
    WORKDIR /opt/services/djangoapp/src

    # install our two dependencies
    RUN pip install gunicorn django

    # copy our project code
    COPY . /opt/services/djangoapp/src

    # expose the port 8000
    EXPOSE 8000

    # define the default command to run when starting the container
    CMD ["gunicorn", "--chdir", "hello", "--bind", ":8000", "hello.wsgi:application"]
    ```
- `docker build . -t hello` --> build image
- `docker run -p 8000:8000 hello` --> run hello container
- http://localhost:8000

## pipfile & Pipfile.lock
- Note: Mengunci dependensi, menggantikan requrements.txt
- `pip3 install pipenv`
- Buat `pipfile` di root:
    ```conf
    [[source]]
    url = "https://pypi.python.org/simple"
    verify_ssl = true
    name = "pypi"


    [packages]
    Django = "*"
    gunicorn = "*"


    [requires]
    # our Dockerfile is based on Python 3.6
    python_version = "3.6"
    ```
- `pipenv lock` --> menghasilkan file Pipfile.lock
- Update dockerfile, agar bisa menginstall dependensi dari pipenv
    ```docker
    # start from an official image
    FROM python:3.8

    # arbitrary location choice: you can change the directory
    RUN mkdir -p /opt/services/djangoapp/src
    WORKDIR /opt/services/djangoapp/src

    # install our dependencies
    # we use --system flag because we don't need an extra virtualenv
    COPY Pipfile Pipfile.lock /opt/services/djangoapp/src/
    RUN pip install pipenv && pipenv install --system

    # copy our project code
    COPY . /opt/services/djangoapp/src

    # expose the port 8000
    EXPOSE 8000

    # define the default command to run when starting the container
    CMD ["gunicorn", "--chdir", "hello", "--bind", ":8000", "hello.wsgi:application"]
    ```
- `docker build . -t hello`
- `docker run -p 8000:8000 hello` --> run hello container
- http://localhost:8000

## Docker Compose - Djagoapp
- Buat `docker-compose.yml` --> tempatkan di root
    ```yml
    version: '3'

    services:
        djangoapp:
            build: .
            volumes:
            - .:/opt/services/djangoapp/src
            ports:
            - 8000:8000
    ```
    - Note:
        - rebuild container djangoapp(Django + Gunicorn) meggunakan compose
        - volume --> path di docker
- `docker-compose up`
- run http://localhost:8000

## Docker Compose - Nginx
- edit `docker-compose.yml`
    ```yml
    version: '3'

    services:

        djangoapp:
            build: .
            volumes:
            - .:/opt/services/djangoapp/src

        nginx:
            image: nginx
            ports:
            - 8000:80
            volumes:
            - ./config/nginx/conf.d:/etc/nginx/conf.d
            depends_on:  # <-- wait for djangoapp to be "ready" before starting this service
            - djangoapp
    ```
    - Note:
        - image menggunakan nginx:latest
        - remove ports pada djangoapp, ports pindah ke nginx, karena kita akan menggunakannya sebagai webservice (tidak lagi gunicorn)
        - 8000:80 --> Nginx diakses dari luar menggunakan port 8000, dan berjalan di dalam docker pada port 80
        - binding folder `/etc/nginx/conf.d` (dalam docker) ke `./config/nginx/conf.d` (folder local)
            - `mkdir -p config/nginx/conf.d` --> membuat folder konfigurasi nginx
            - `touch config/nginx/conf.d/local.conf` --> membuat file konfigurasi nginx
        - depends on --> nginx run, Menunggu djangoapp siap dulu 
- edit `config/nginx/conf.d/local.conf`
    ```conf
    # first we declare our upstream server, which is our Gunicorn application
    upstream hello_server {
        # docker will automatically resolve this to the correct address
        # because we use the same name as the service: "djangoapp"
        server djangoapp:8000;
    }

    # now we declare our main server
    server {

        listen 80;
        server_name localhost;

        location / {
            # everything is passed to Gunicorn
            proxy_pass http://hello_server;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_redirect off;
        }
    }
    ```

## Nginx <--bridge--> djangoapp
- edit `docker-compose.yml`
    ```yml
    version: '3'

    services:

    djangoapp:
        build: .
        volumes:
        - .:/opt/services/djangoapp/src
        networks:  # <-- here
        - nginx_network

    nginx:
        image: nginx:1.13
        ports:
        - 8000:80
        volumes:
        - ./config/nginx/conf.d:/etc/nginx/conf.d
        depends_on:
        - djangoapp
        networks:  # <-- here
        - nginx_network

    networks:  # <-- and here
    nginx_network:
        driver: bridge
    ```
- `docker-compose up`
- run http://localhost:8000

## Docker Compose - postgreSQL
- Tujuan: Merubah db default django dari sqlite ke posgresql
- Install psycopg2 via Pipfile
    - edit Pipfile
        ```
        [[source]]
        url = "https://pypi.python.org/simple"
        verify_ssl = true
        name = "pypi"


        [packages]
        Django = "*"
        gunicorn = "*"
        "psycopg2" = "*"


        [requires]
        # our Dockerfile is based on Python 3.6
        python_version = "3.6"
        ```

    - `pipenv lock`
    - `docker-compose build`

- Edit django setting
    ```conf
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': 'database1',
            'USER': 'database1_role',
            'PASSWORD': 'database1_password',
            'HOST': 'database1',  # <-- IMPORTANT: same name as docker-compose service!
            'PORT': '5432',
        }
    }
    ```
- File environment/config db
    - Buat config/db/database1_env
        ```
        mkdir config/db
        touch config/db/database1_env
        ```
    - edit config/db/database1_env
        ```
        POSTGRES_USER=database1_role
        POSTGRES_PASSWORD=database1_password
        POSTGRES_DB=database1
        ```
- Tambahkan service postgres di `docker-compose.yml`
    ```yml
    version: '3'

    services:

    djangoapp:
        build: .
        volumes:
        - .:/opt/services/djangoapp/src
        networks:
        - nginx_network
        - database1_network  # <-- connect to the bridge
        depends_on:  # <-- wait for db to be "ready" before starting the app
        - database1

    nginx:
        image: nginx
        ports:
        - 8000:80
        volumes:
        - ./config/nginx/conf.d:/etc/nginx/conf.d
        depends_on:
        - djangoapp
        networks:
        - nginx_network

    database1:  # <-- IMPORTANT: same name as in DATABASES setting, otherwise Django won't find the database!
        image: postgres
        env_file:  # <-- we use the previously defined values
        - config/db/database1_env
        networks:  # <-- connect to the bridge
        - database1_network
        volumes:
        - database1_volume:/var/lib/postgresql/data

    networks:
    nginx_network:
        driver: bridge
    database1_network:  # <-- add the bridge
        driver: bridge

    volumes:
    database1_volume:
    ```
    - Note:
        - database1 --> nama service harus sama dengan setting django dan db config
        - env_file --> menghubungkan ke file env db
        - networks: --> menghubungkan ke network baru, database1_network
        - volumes --> binding database1_volume (local) dari /var/lib/postgresql/data (docker)
        - Membuat root database1_network
        - Membuat root database1_volume

- `docker-compose build`
- `docker-compose run --rm djangoapp /bin/bash -c "cd hello; ./manage.py migrate"`

## Static Files
- staticfile di dockerfile tidak disarankan.disarankan menjalankan manual collectstatic dari dalam docker `docker-compose run djangoapp hello/manage.py collectstatic --no-input`
- Django project setting
    ```conf
    # as declared in NginX conf, it must match /opt/services/djangoapp/static/
    STATIC_ROOT = os.path.join(os.path.dirname(os.path.dirname(BASE_DIR)), 'static')

    # do the same for media files, it must match /opt/services/djangoapp/media/
    MEDIA_ROOT = os.path.join(os.path.dirname(os.path.dirname(BASE_DIR)), 'media')
    ```
- Edit `config/nginx/conf.d/local.conf`
    ```conf
    upstream hello_server {
        server djangoapp:8000;
    }

    server {

        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://hello_server;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_redirect off;
        }

        location /static/ {
            alias /opt/services/djangoapp/static/;
        }

        location /media/ {
            alias /opt/services/djangoapp/media/;
        }
    }
    ```
- Tambahkan volume di `docker-compose.yml`
    ```yml
    version: '3'

    services:

    djangoapp:
        build: .
        volumes:
        - .:/opt/services/djangoapp/src
        - static_volume:/opt/services/djangoapp/static  # <-- bind the static volume
        - media_volume:/opt/services/djangoapp/media  # <-- bind the media volume
        networks:
        - nginx_network
        - database1_network
        depends_on:
        - database1

    nginx:
        image: nginx:1.13
        ports:
        - 8000:80
        volumes:
        - ./config/nginx/conf.d:/etc/nginx/conf.d
        - static_volume:/opt/services/djangoapp/static  # <-- bind the static volume
        - media_volume:/opt/services/djangoapp/media  # <-- bind the media volume
        depends_on:
        - djangoapp
        networks:
        - nginx_network

    database1:
        image: postgres:10
        env_file:
        - config/db/database1_env
        networks:
        - database1_network
        volumes:
        - database1_volume:/var/lib/postgresql/data

    networks:
    nginx_network:
        driver: bridge
    database1_network:
        driver: bridge

    volumes:
    database1_volume:
    static_volume:  # <-- declare the static volume
    media_volume:  # <-- declare the media volume
    ```

- Jalankan `docker-compose run djangoapp hello/manage.py collectstatic --no-input`