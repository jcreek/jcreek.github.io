---
tags:
  - deploying
  - docker
---

# Docker Compose for S3 Backup and Restore of PostgreSQL

_2021-07-12_

In this post, I will cover how you can set up two Docker containers for backing up and restoring Postgres databases using S3 in AWS.

The complete `docker-compose.yml` file looks like this. Note that it's set to link these containers to a database container also configured within the same compose file - this could be modified to connect to an external database if desired by removing the `links` and changing the `POSTGRES_HOST` environment variables.

```docker
version: '3.4'

services:
    pgbackups3:
        build:
            context: .
            dockerfile: postgres-backup-s3/Dockerfile
        links:
            - db
        environment:
            SCHEDULE: '@daily'
            S3_REGION: eu-west-2
            S3_ACCESS_KEY_ID: keygoeshere
            S3_SECRET_ACCESS_KEY: secretkeygoeshere
            S3_BUCKET: yourapp-backups
            S3_PREFIX: backup
            POSTGRES_HOST: db
            POSTGRES_DATABASE: yourdbname
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: passwordgoeshere
            POSTGRES_EXTRA_OPTS: '--schema=public --blobs'    
    pgrestores3:
        build:
            context: .
            dockerfile: postgres-restore-s3/Dockerfile
        links:
            - db
        environment:
            S3_ACCESS_KEY_ID: keygoeshere
            S3_SECRET_ACCESS_KEY: secretkeygoeshere
            S3_BUCKET: yourapp-backups
            S3_PREFIX: backup
            POSTGRES_HOST: db
            POSTGRES_DATABASE: yourdbname
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: passwordgoeshere
            DROP_PUBLIC: 'yes'
```

## Backing up PostgreSQL to S3

The Dockerfile used looks like this:

```docker
FROM alpine:3.13

RUN apk update \
	&& apk add coreutils \
	&& apk add postgresql-client \
	&& apk add python3 py3-pip && pip3 install --upgrade pip && pip3 install awscli \
	&& apk add openssl \
	&& apk add curl \
	&& curl -L --insecure https://github.com/odise/go-cron/releases/download/v0.0.6/go-cron-linux.gz | zcat > /usr/local/bin/go-cron && chmod u+x /usr/local/bin/go-cron \
	&& apk del curl \
	&& rm -rf /var/cache/apk/*

ENV POSTGRES_DATABASE **None**
ENV POSTGRES_HOST **None**
ENV POSTGRES_PORT 5432
ENV POSTGRES_USER **None**
ENV POSTGRES_PASSWORD **None**
ENV POSTGRES_EXTRA_OPTS ''
ENV S3_ACCESS_KEY_ID **None**
ENV S3_SECRET_ACCESS_KEY **None**
ENV S3_BUCKET **None**
ENV S3_REGION us-west-1
ENV S3_PATH 'backup'
ENV S3_ENDPOINT **None**
ENV S3_S3V4 no
ENV SCHEDULE **None**

COPY ["postgres-backup-s3/run.sh", "run.sh"]
COPY ["postgres-backup-s3/backup.sh", "backup.sh"]

CMD ["sh", "run.sh"]
```

This is a combination of parts from [here](https://github.com/itbm/postgresql-backup-s3/blob/master/Dockerfile) and the original [here](https://github.com/schickling/dockerfiles/tree/master/postgres-backup-s3) with some further customisation by me.

It essentially sets up a Linux environment to be able to connect to both S3 and Postgres, and to be able to run two script files.

### `run.sh`

This one handles a bit of aws configuration for S3 connections, as well as setting up the behaviour for scheduling or just immediately running the container. I like to set it to run daily so I can set it and forget it.

```bash
#! /bin/sh

set -e

if [ "${S3_S3V4}" = "yes" ]; then
    aws configure set default.s3.signature_version s3v4
fi

if [ "${SCHEDULE}" = "**None**" ]; then
  sh backup.sh
else
  exec go-cron "$SCHEDULE" /bin/sh backup.sh
fi
```

### `backup.sh`

This one handles checking all the requisite connection details are present, then makes a compressed dump file of the PostgreSQL database, and uploads it to S3.

```bash
#! /bin/sh

set -e
set -o pipefail

if [ "${S3_ACCESS_KEY_ID}" = "**None**" ]; then
  echo "You need to set the S3_ACCESS_KEY_ID environment variable."
  exit 1
fi

if [ "${S3_SECRET_ACCESS_KEY}" = "**None**" ]; then
  echo "You need to set the S3_SECRET_ACCESS_KEY environment variable."
  exit 1
fi

if [ "${S3_BUCKET}" = "**None**" ]; then
  echo "You need to set the S3_BUCKET environment variable."
  exit 1
fi

if [ "${POSTGRES_DATABASE}" = "**None**" ]; then
  echo "You need to set the POSTGRES_DATABASE environment variable."
  exit 1
fi

if [ "${POSTGRES_HOST}" = "**None**" ]; then
  if [ -n "${POSTGRES_PORT_5432_TCP_ADDR}" ]; then
    POSTGRES_HOST=$POSTGRES_PORT_5432_TCP_ADDR
    POSTGRES_PORT=$POSTGRES_PORT_5432_TCP_PORT
  else
    echo "You need to set the POSTGRES_HOST environment variable."
    exit 1
  fi
fi

if [ "${POSTGRES_USER}" = "**None**" ]; then
  echo "You need to set the POSTGRES_USER environment variable."
  exit 1
fi

if [ "${POSTGRES_PASSWORD}" = "**None**" ]; then
  echo "You need to set the POSTGRES_PASSWORD environment variable or link to a container named POSTGRES."
  exit 1
fi

if [ "${S3_ENDPOINT}" == "**None**" ]; then
  AWS_ARGS=""
else
  AWS_ARGS="--endpoint-url ${S3_ENDPOINT}"
fi

# env vars needed for aws tools
export AWS_ACCESS_KEY_ID=$S3_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=$S3_SECRET_ACCESS_KEY
export AWS_DEFAULT_REGION=$S3_REGION

export PGPASSWORD=$POSTGRES_PASSWORD
POSTGRES_HOST_OPTS="-h $POSTGRES_HOST -p $POSTGRES_PORT -U $POSTGRES_USER $POSTGRES_EXTRA_OPTS"

echo "Creating dump of ${POSTGRES_DATABASE} database from ${POSTGRES_HOST}..."

pg_dump $POSTGRES_HOST_OPTS $POSTGRES_DATABASE | gzip > dump.sql.gz

echo "Uploading dump to $S3_BUCKET"

cat dump.sql.gz | aws $AWS_ARGS s3 cp - s3://$S3_BUCKET/$S3_PREFIX/${POSTGRES_DATABASE}_$(date +"%Y-%m-%dT%H:%M:%SZ").sql.gz || exit 2

echo "SQL backup uploaded successfully"

```

## Restoring PostgreSQL from S3

The Dockerfile used looks like this:

```docker
FROM alpine:3.13

RUN apk update \
	&& apk add coreutils \
	&& apk add postgresql-client \
	&& apk add python3 py3-pip && pip3 install --upgrade pip && pip3 install awscli \
	&& apk add openssl \
	&& apk add curl \
	&& curl -L --insecure https://github.com/odise/go-cron/releases/download/v0.0.6/go-cron-linux.gz | zcat > /usr/local/bin/go-cron && chmod u+x /usr/local/bin/go-cron \
	&& apk del curl \
	&& rm -rf /var/cache/apk/*

ENV POSTGRES_DATABASE **None**
ENV POSTGRES_HOST **None**
ENV POSTGRES_PORT 5432
ENV POSTGRES_USER **None**
ENV POSTGRES_PASSWORD **None**
ENV S3_ACCESS_KEY_ID **None**
ENV S3_SECRET_ACCESS_KEY **None**
ENV S3_BUCKET **None**
ENV S3_REGION us-west-1
ENV S3_PATH 'backup'
ENV DROP_PUBLIC 'no'

COPY ["postgres-restore-s3/restore.sh", "restore.sh"]

CMD ["sh", "restore.sh"]

```

This is once again a combination of parts from [here](https://github.com/itbm/postgresql-backup-s3/blob/master/Dockerfile) and the original [here](https://github.com/schickling/dockerfiles/tree/master/postgres-backup-s3) with some further customisation by me.

It essentially sets up a Linux environment to be able to connect to both S3 and Postgres, and to be able to run one script file.

### `restore.sh`

This one handles checking all the requisite connection details are present, then retrieves the most recent compressed dump file of the PostgreSQL database from S3, drops everything in the public space in PostgreSQL, and restores the backup. Crucially it then deletes the downloaded backup dump file to enable running the same container again for future restores without issue.

```bash
#! /bin/sh

set -e
set -o pipefail

if [ "${S3_ACCESS_KEY_ID}" = "**None**" ]; then
  echo "You need to set the S3_ACCESS_KEY_ID environment variable."
  exit 1
fi

if [ "${S3_SECRET_ACCESS_KEY}" = "**None**" ]; then
  echo "You need to set the S3_SECRET_ACCESS_KEY environment variable."
  exit 1
fi

if [ "${S3_BUCKET}" = "**None**" ]; then
  echo "You need to set the S3_BUCKET environment variable."
  exit 1
fi

if [ "${POSTGRES_DATABASE}" = "**None**" ]; then
  echo "You need to set the POSTGRES_DATABASE environment variable."
  exit 1
fi

if [ "${POSTGRES_HOST}" = "**None**" ]; then
  if [ -n "${POSTGRES_PORT_5432_TCP_ADDR}" ]; then
    POSTGRES_HOST=$POSTGRES_PORT_5432_TCP_ADDR
    POSTGRES_PORT=$POSTGRES_PORT_5432_TCP_PORT
  else
    echo "You need to set the POSTGRES_HOST environment variable."
    exit 1
  fi
fi

if [ "${POSTGRES_USER}" = "**None**" ]; then
  echo "You need to set the POSTGRES_USER environment variable."
  exit 1
fi

if [ "${POSTGRES_PASSWORD}" = "**None**" ]; then
  echo "You need to set the POSTGRES_PASSWORD environment variable or link to a container named POSTGRES."
  exit 1
fi

# env vars needed for aws tools
export AWS_ACCESS_KEY_ID=$S3_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=$S3_SECRET_ACCESS_KEY
export AWS_DEFAULT_REGION=$S3_REGION

export PGPASSWORD=$POSTGRES_PASSWORD
POSTGRES_HOST_OPTS="-h $POSTGRES_HOST -p $POSTGRES_PORT -U $POSTGRES_USER"

echo "Finding latest backup"

LATEST_BACKUP=$(aws s3 ls s3://$S3_BUCKET/$S3_PREFIX/ | sort | tail -n 1 | awk '{ print $4 }')

echo "Fetching ${LATEST_BACKUP} from S3"

aws s3 cp s3://$S3_BUCKET/$S3_PREFIX/${LATEST_BACKUP} dump.sql.gz
gzip -d dump.sql.gz

if [ "${DROP_PUBLIC}" == "yes" ]; then
	echo "Recreating the public schema"
	psql $POSTGRES_HOST_OPTS -d $POSTGRES_DATABASE -c "drop schema public cascade; create schema public;"
fi

echo "Restoring ${LATEST_BACKUP}"

psql $POSTGRES_HOST_OPTS -d $POSTGRES_DATABASE < dump.sql

echo "Restore complete"

rm -f ./dump.sql

echo "Deleted dump files"
```
