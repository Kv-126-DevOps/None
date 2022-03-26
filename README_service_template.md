# SERVICE_NAME

## Overview

Brief service description

## Prerequisites

* Python 3.6+
* PIP package manager
* Database for data storage
* etc

## Running the Application

1. Clone the repository and go to the application folder

   ```bash
   git clone --branch develop https://github.com/Kv-126-DevOps/app-example.git /opt/app-example
   cd /opt/app-example
   ```

1. Install all required python packages

   ```bash
   pip install -r requirements.txt
   ```
2. Run the application

   ```bash
   export POSTGRES_HOST=postgres.dev.net
   export POSTGRES_USER=dbuser
   export POSTGRES_PW=dbpass
   flask run --host=0.0.0.0
   ```

### Application Properties

Parameters are set as environment variables

| Parameter     | Default     | Description     |
|:--------------|:------------|:----------------|
| POSTGRES_HOST | `localhost` | PostgreSQL host |

## Building

1. Create a docker image

   ```bash
   docker build . --no-cache -t app-example:0.0.1
   ```

2. Push a docker image to a docker repository

   ```bash
   docker push app-example:0.0.1
   ```

## Testing

1. Do unit tests

   ```bash
   python unittests.py
   ```
