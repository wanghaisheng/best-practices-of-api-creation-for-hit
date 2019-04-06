

Ubuntu Server 16.04
===================

### Overview (WIP)[¶](#overview-wip "Permanent link")

You can deploy the subZero stack on any linux box, we'll install all components on a single Ubuntu Server 16.04 for this example.

Go to the root of your project(we're using `postgrest-starter-kit`) and connect to your server like this:

\## Add a PRODUCTION\_DB\_HOST env var
local> echo "PRODUCTION\_DB\_HOST=example.com" >> .env && source .env
\## We'll source the project env vars in the host
local> scp .env ubuntu@$PRODUCTION\_DB\_HOST:/tmp/
local> ssh ubuntu@$PRODUCTION\_DB\_HOST
\## We need to run several privileged commands so we will go with sudo from here
ubuntu> sudo su
\## source env vars and delete .env file
ubuntu> source /tmp/.env && rm /tmp/.env

Now we'll install the components on the server.

### Install PostgreSQL 9.6[¶](#install-postgresql-96 "Permanent link")

Using the instructions in [https://www.postgresql.org/download/linux/ubuntu](https://www.postgresql.org/download/linux/ubuntu), we'll run:

ubuntu> wget -qO - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
ubuntu> add-apt-repository "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb\_release -sc)\-pgdg main"
ubuntu> apt-get update
ubuntu> apt-get install -y postgresql-9.6

Create your db superuser(we will use this role for running migrations), your application database and authenticator role:

ubuntu> sudo -u postgres psql << EOF
create role $SUPER\_USER with login password '$SUPER\_USER\_PASSWORD' superuser;
create database $DB\_NAME;
create role $DB\_USER with login password '$DB\_PASS';
EOF

Allow all tcp/ip connections:

ubuntu> echo "listen\_addresses = '\*'" >> /etc/postgresql/9.6/main/postgresql.conf
ubuntu> echo "host all all 0.0.0.0/0 md5" >> /etc/postgresql/9.6/main/pg\_hba.conf
ubuntu> systemctl restart postgresql

Check if you can connect to the host postgresql server:

local> psql postgres://$SUPER\_USER:$SUPER\_USER\_PASSWORD@$PRODUCTION\_DB\_HOST/$DB\_NAME -c "SELECT 1"

If there's any error check if your firewall rules allow connecting to port 5432.

### Install PostgREST[¶](#install-postgrest "Permanent link")

We'll download the latest release for PostgREST, create its config file and a systemd unit file.

ubuntu> wget -qO- http://github.com/begriffs/postgrest/releases/download/v0.4.4.0/postgrest-v0.4.4.0-ubuntu.tar.xz | tar xvJ -C /bin/

ubuntu> mkdir /etc/postgrest && cat > /etc/postgrest/config <<EOF
db-uri = "postgres://$DB\_USER:$DB\_PASS@localhost:5432/$DB\_NAME"
db-schema = "$DB\_SCHEMA"
db-anon-role = "$DB\_ANON\_ROLE"
db-pool = 10

server-host = "127.0.0.1"
server-port = 3000

jwt-secret = "$JWT\_SECRET"
EOF

ubuntu> cat > /etc/systemd/system/postgrest.service <<\\EOF
\[Unit\]
Description=REST API for any Postgres database
After=postgresql.service

\[Service\]
ExecStart=/bin/postgrest /etc/postgrest/config
ExecReload=/bin/kill -HUP $MAINPID

\[Install\]
WantedBy=multi-user.target
EOF

Next we'll enable the service to be available at boot time, start it and make a test call:

ubuntu> systemctl enable postgrest
ubuntu> systemctl start postgrest
ubuntu> curl -v localhost:3000
\# It'll respond with:
\# {"hint":null,"details":null,"code":"22023","message":"role \\"anonymous\\" does not exist"}

Don't worry about the error message, it'll be cleared out when we deploy the first migration.

### Install openresty[¶](#install-openresty "Permanent link")

Using the instructions for Ubuntu in [https://openresty.org/en/linux-packages.html](https://openresty.org/en/linux-packages.html):

ubuntu> wget -qO - https://openresty.org/package/pubkey.gpg | apt-key add -
ubuntu> add-apt-repository -y "deb http://openresty.org/package/ubuntu $(lsb\_release -sc) main"
ubuntu> apt-get update
ubuntu> apt-get install -y openresty

You should be able to get the openresty welcome page from your server with:

local> curl -v $PRODUCTION\_DB\_HOST

Check your firewall rules for port 80 if there's no successful response. We'll need to run a few more commands for the postgrest-starter-kit to work:

ubuntu> echo "127.0.0.1 postgrest" >> /etc/hosts
ubuntu> echo -e '\[Service\]\\nEnvironment="POSTGREST\_HOST=127.0.0.1"\\nEnvironment="POSTGREST\_PORT=3000"' | SYSTEMD\_EDITOR\=tee systemctl edit openresty
ubuntu> systemctl restart openresty

Now all components are ready for deployment.

### Deploy postgrest starter kit[¶](#deploy-postgrest-starter-kit "Permanent link")

Create a `subzero-app.json` file like this:

local> source .env && SUBZERO\_APP\_CONF\=$(cat <<EOF
{
 "name": "$COMPOSE\_PROJECT\_NAME",
 "openresty\_image\_type": "default",
 "db\_location": "external",
 "db\_host": "$PRODUCTION\_DB\_HOST",
 "db\_port": 5432,
 "db\_name": "$DB\_NAME"
}
EOF
)

local> echo "${SUBZERO\_APP\_CONF}" > subzero-app.json

Then we'll deploy the migrations and reload the postgrest service with:

local> subzero cloud app-deploy --dba $SUPER\_USER --password $SUPER\_USER\_PASSWORD
\## You should see something like:
\## Checking PostgreSQL connection info (postgres://dbadmin@example.com:5432/db)
\## Skipping OpenResty image building
\## Deploying migrations with sqitch
\## Adding registry tables to db:pg://dbadmin:@example.com:5432/app
\## Deploying changes to db:pg://dbadmin:@example.com:5432/db
\##  + 0000000001-initial .. ok

local> ssh ubuntu@$PRODUCTION\_DB\_HOST <<EOF
 sudo systemctl reload postgrest
EOF

For the last part, we'll copy openresty configs, lua files and reload the openresty service:

local> scp openresty/nginx/conf/nginx.conf ubuntu@$PRODUCTION\_DB\_HOST:/tmp
local> scp -r openresty/nginx/conf/includes ubuntu@$PRODUCTION\_DB\_HOST:/tmp
local> scp -r openresty/nginx/html ubuntu@$PRODUCTION\_DB\_HOST:/tmp
local> scp -r openresty/lualib/user\_code ubuntu@$PRODUCTION\_DB\_HOST:/tmp

local> ssh -q ubuntu@$PRODUCTION\_DB\_HOST <<EOF
 sudo su
 mv /tmp/nginx.conf /usr/local/openresty/nginx/conf/nginx.conf
 mkdir -p /usr/local/openresty/nginx/conf/includes/ && mv /tmp/includes/\* /usr/local/openresty/nginx/conf/includes/
 mv /tmp/html/\* /usr/local/openresty/nginx/html/
 mkdir -p /usr/local/openresty/lualib/user\_code/ && mv /tmp/user\_code/\* /usr/local/openresty/lualib/user\_code/
 systemctl reload openresty
EOF

Now run the command:

curl -v http://$PRODUCTION\_DB\_HOST/rest/todos?select\=id,todo
\# It should give: \[\]

### Done![¶](#done "Permanent link")
