Matrix / Synapse on digital ocean with dokku
============================================

Creating the dokku droplet and prepare everything for Synapse
-------------------------------------------------------------

Ceate a [dokku][dokku] droplet from [digital ocean marketplace][domarket] and
finish the set-up the droplet / dokku install according to the readme

Add a DNS entry for the newly created droplet and also an entry for the synapse
server pointing to the same IP. If you'd also like to install the
[Element-Web][ew] as a web-UI, add another entry for this service.

For example:

 | record type | domain             | value             | remarks                                    |
 | ----------- | ------------------ | ----------------- | ------------------------------------------ |
 | A           | dokku.example.org  | <an ipv4 addess>  | used for connecting to the droplet         |
 | CNAME       | matrix.example.org | dokku.example.org | the [Synapse][syn] server, matrix protocol |
 | CNAME       | chat.example.org   | dokku.example.org | the Element-Web frontend                   |


Connect to the droplet via ssh:

    #First step: make updates

    remote> apt-get update
    remote> apt-get --with-new-pkgs upgrade
    remote> apt-get autoremove
    remote> apt-get autoclean
    remote> reboot

Reconnect to the droplet via ssh:

    # add port 8448 to the firewall, this is needed for the matrix federation
    remote> ufw allow 8448/tcp

    # create the application
    # using 'matrix' as a name here for the correct virtual host entry later
    remote> dokku apps:create matrix

    # create a directory for persitent storage
    remote> mkdir -p /var/lib/dokku/data/storage/synapse-data

    # ensure the proper user has access to this directory
    remote> chown -R dokku:dokku /var/lib/dokku/data/storage/synapse-data

    # mount the directory into your container's /data directory
    remote> dokku storage:mount matrix /var/lib/dokku/data/storage/synapse-data:/data

    # install the dokku postgres plugin
    remote> dokku plugin:install https://github.com/dokku/dokku-postgres.git


The next thing is **scary**

We need to set the locale for a new postgres database to "C" to for synapse.
Unfortunately I didn't find a way to pass this somehow to the
`dokku postgres:create`, so we need to modify the plugin script, create the
database, check if it worked and remove the modification.

    # uncomment one line and add a slightly modified one below

    remote> vim /var/lib/dokku/plugins/enabled/postgres/functions

    [...]
    #docker exec "$SERVICE_NAME" su - postgres -c "createdb -E utf8 $DATABASE_NAME" 2>/dev/null || dokku_log_verbose_quiet 'Already exists
    docker exec "$SERVICE_NAME" su - postgres -c "createdb -E utf8 --locale=C -T template0 $DATABASE_NAME" 2>/dev/null || dokku_log_verbose_quiet 'Already exists'
    [...]

    # create a postgres service with the name synapsedb
    remote> dokku postgres:create synapsedb

    #check if the datatbase was created successfully with the "C" locale
    remote> dokku postgres:enter synapsedb
    remote#synapsedb> psql -U postgres
    remote#postgres=# select datname, datcollate, datctype from pg_database;

    # restore the original postgres createdb command
    remote> vim /var/lib/dokku/plugins/enabled/postgres/functions
    [...]
    docker exec "$SERVICE_NAME" su - postgres -c "createdb -E utf8 $DATABASE_NAME" 2>/dev/null || dokku_log_verbose_quiet 'Already exists
    #docker exec "$SERVICE_NAME" su - postgres -c "createdb -E utf8 --locale=C -T template0 $DATABASE_NAME" 2>/dev/null || dokku_log_verbose_quiet 'Already exists'
    [...]

After the change is reverted, we can now link the database to the application

    remote> dokku postgres:link synapsedb matrix

    # get the info on how to reach the database
    remote> dokku config:get matrix DATABASE_URL
    postgres://postgres:random_postgres_password@dokku-postgres-synapsedb:5432/synapsedb


Before we leave the remote, note the uid and gid for the dokku user and group - we'll need it in the next step

    remote> id dokku
    uid=1000(dokku) gid=1000(dokku) [...]


Setting up the config files for Synapse
---------------------------------------

Run synapse in docker locally to create the config files
and the private signing key in the image. Replace the UID and GID parameters
with the one from above and of cause adjust your server name ;-)

    local> docker run -it --rm \
    --mount type=volume,src=synapse-data,dst=/data \
    --mount type=volume,src=synapse-data,dst=/config \
    -e SYNAPSE_SERVER_NAME=matrix.example.org \
    -e SYNAPSE_REPORT_STATS=yes \
    -e SYNAPSE_CONFIG_DIR=/config \
    -e UID=1000 \
    -e GID=1000 \
    matrixdotorg/synapse:latest generate

Create a new directory with a git repo and add the files created in the
previous step: `homeserver.yaml`, `matrix.example.com.log.config` and
`maxtrix.example.com.signing.key`

Modify the `config_data/homeserver.yaml` on the local machine to reflect the
data from the postgres info:

    [...]
    # Example Postgres configuration:
    #
    database:
      name: psycopg2
      args:
        user: postgres
        password: random_postgres_password
        database: synapsedb
        host: dokku-postgres-synapsedb
        port: 5432
        cp_min: 5
        cp_max: 10
    #
    # For more information on using Synapse with Postgres, see `docs/postgres.md`.
    #
    # database:
    #   name: sqlite3
    #  args:
    #    database: /data/homeserver.db
    [...]

add the public base url to the file:

    public_baseurl: https://matrix.example.com/

and also add the basic mail settings in section `email`:

    # The hostname of the outgoing SMTP server to use. Defaults to 'localhost'.
    smtp_host: mail.example.com

    # The port on the mail server for outgoing SMTP. Defaults to 25.
    smtp_port: 587

    # Username/password for authentication to the SMTP server. By default, no
    # authentication is attempted.
    smtp_user: "exampleusername"
    smtp_pass: "examplepassword"

    # Uncomment the following to require TLS transport security for SMTP.
    # By default, Synapse will connect over plain text, and will then switch to
    # TLS via STARTTLS *if the SMTP server supports it*. If this option is set,
    # Synapse will refuse to connect unless the server supports STARTTLS.
    #
    require_transport_security: true

    # notif_from defines the "From" address to use when sending emails.
    # It must be set if email sending is enabled.
    #
    # The placeholder '%(app)s' will be replaced by the application name,
    # which is normally 'app_name' (below), but may be overridden by the
    # Matrix client application.
    #
    # Note that the placeholder must be written '%(app)s', including the
    # trailing 's'.
    #
    notif_from: "Your Friendly %(app)s homeserver <noreply@example.com>"


Also adjust the log file path in `config_data/matrix.example.com.log.config`:

     filename: /data/homeserver.log

Now a dockerfile is needed for dokku: create a new file in the local git repo
named "Dockerfile" with the following content:

    # reference the latest synapse release
    FROM matrixdotorg/synapse:latest

    # set environment variables
    ENV SYNAPSE_SERVER_NAME matrix.example.org
    ENV SYNAPSE_REPORT_STATS yes
    ENV SYNAPSE_CONFIG_DIR /config
    ENV UID 1000
    ENV GID 1000

    # copy the right files to the right place
    ADD nginx.conf.sigil /
    ADD --chown=1000:1000 homeserver.yaml /config/
    ADD --chown=1000:1000 matrix.example.org.log.config /config/
    ADD --chown=1000:1000 matrix.example.org.signing.key /config/

Theoretically everything should be ok to deploy the repo to dokku for the first
time. But unfortunately one more file is missing in the repo.

For matrix to work nicely, the server on port 8448 must be considered as
"default_server" in the nginx config. Dokku uses "sigil" for templating and the
easiest would be to copy the file from this repo – I also added a ".well-known"
URL for matrix.


Deploying synapse with dokku
----------------------------

To deploy an app with dokku, xou just have to push a git repo to the right server url:

    # add dokku as remote
    local> git remote add dokku dokku@dokku.example.com:/matrix

    local> git add . && git commit -m "first deploy"

    # push the dockerfile
    local> git push dokku master

**WARNING**: Do not push a dokku deploy repo to a public repo since its config files might contain sensitive data like keys and passwords.

For the next step - acquiring a [let's encrypt][lets] certificate - we need a
working http server on port 80. therefore we'll forward the requests to port
8008. Nginx will automatically forward http to https requests as soon as we
have a certificate.

    # add a proxy on port 80
    remote> dokku proxy:ports-add matrix http:80:8008


Let's add let's encrypt

    # add let's encrypt dokku plugin
    remote> dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git

    # setup let's  encrypt for synapse
    remote> dokku config:set --no-restart matrix DOKKU_LETSENCRYPT_EMAIL=your@email.tld
    remote> dokku letsencrypt:enable matrix

    # and enable automatic autorenewal
    remote> dokku letsencrypt:cron-job --add

After Let's encrypt works, we need to adjust port 8448 from http to https

    # switch proxy port 8448 to https
    remote> dokku proxy:ports-remove matrix 8448
    remote> dokku proxy:ports-add matrix https:8448:8008


If you now visit http://matrix.example.org you should see something \o/


Let's create the first user:

    remote> dokku enter matrix
    remote#matrix> register_new_matrix_user -c /config/homeserver.yaml http://localhost:8008

This will prompt you for the new user details:

    New user localpart [root]: johndoe
    Password:
    Confirm password:
    Make admin [no]: yes
    Sending registration request...
    Success!

Hopfully, the last line is right...

If you visit https://matrix.example.org you should see a welcome page from
Synapse.


Optional: add Element-Web as web frontend to synapse
----------------------------------------------------

This is so much less work than getting synapse to run - some of the
foundational work was alredy done.

First we need a new / second local git repo for Synapse. the repo just needs
two files: a docker file für dokku and a `config.json`.

The docker file is quite simple:

    FROM vectorim/element-web

    ADD config.json /app/config.json

As for the config file, there is an example file available at
https://github.com/vector-im/element-web/blob/develop/config.sample.json
that just needs some minor adjustments (and a renaming to `config.json`):

The most important thing is to set the base url and server name.

    "base_url": "https://matrix.example.org",
    "server_name": "matrix.example.org"

The [documentation][edoc] for the config file is quite extensive, make
adjustments as you like.

everything should be prepared now to deploy Element-Web to dokku. As a first step add a new dokku app:

    # create the deployment app
    # the name will also be the virtual host name later - here "chat" is used
    # (take a look again at the top of the readme at the dns entries)
    remote> dokku apps:create chat

The next step is to deploy the app:

    # add dokku as remote
    local> git remote add dokku dokku@dokku.example.com:/chat

    local> git add . && git commit -m "first deploy"

    # push the dockerfile
    local> git push chat master

Only one thing is left: add a Let's Encrypt certificate.

    # setup let's  encrypt for element web
    remote> dokku config:set --no-restart chat DOKKU_LETSENCRYPT_EMAIL=your@email.tld
    remote> dokku letsencrypt:enable chat

If you now visit https://chat.example.org, you should be able to log in to your
matrix / synapse server and start chatting \o/.



[ew]: https://matrix.org/docs/projects/client/element
[domarket]: https://marketplace.digitalocean.com/apps/dokku
[dokku]: https://dokku.com
[lets]: https://letsencrypt.org
[syn]: https://github.com/matrix-org/synapse/
[edoc]: https://github.com/vector-im/element-web/blob/develop/docs/config.md
