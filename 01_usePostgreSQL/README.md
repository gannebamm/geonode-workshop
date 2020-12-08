# Install PostgreSQL as backend and work with .env files in PyCharm CE

Since using the sqlite database as backend for development can create issues we will install and configure
a PostgreSQL database alongside our development environment.

## install PostgreSQL with PostGIS extension

The below section is from the [official documentation](https://docs.geonode.org/en/master/install/advanced/core/index.html#postgis-database-setup)

We will add the repository to our ppa list, get the corresponding certificate and install the needed packages. You
can use whichever shell you like, but I would just use the PyCharm integrated one
```shell
# Ubuntu 20.04 (focal)
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
sudo wget --no-check-certificate --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update -y; sudo apt install -y postgresql-13 postgresql-13-postgis-3 postgresql-13-postgis-3-scripts postgresql-13 postgresql-client-13
```

## configure the dbms and create users and dbs

We now must create two databases, geonode and geonode_data, belonging to the role geonode.

```shell
sudo service postgresql start

# Use the password: geonode
sudo -u postgres createuser -P geonode
```

Create database `geonode` and `geonode_data` with owner geonode
```shell
sudo -u postgres createdb -O geonode geonode
sudo -u postgres createdb -O geonode geonode_data
```

Next let’s create PostGIS extensions
```shell
sudo -u postgres psql -d geonode -c 'CREATE EXTENSION postgis;'
sudo -u postgres psql -d geonode -c 'GRANT ALL ON geometry_columns TO PUBLIC;'
sudo -u postgres psql -d geonode -c 'GRANT ALL ON spatial_ref_sys TO PUBLIC;'
sudo -u postgres psql -d geonode -c 'GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO geonode;'

sudo -u postgres psql -d geonode_data -c 'CREATE EXTENSION postgis;'
sudo -u postgres psql -d geonode_data -c 'GRANT ALL ON geometry_columns TO PUBLIC;'
sudo -u postgres psql -d geonode_data -c 'GRANT ALL ON spatial_ref_sys TO PUBLIC;'
sudo -u postgres psql -d geonode_data -c 'GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO geonode;'
```

let us change the connection settings

```shell
sudo nano /etc/postgresql/13/main/pg_hba.conf
```

and change the settings to trust local connections and use md5 for host based access:

```shell
# DO NOT DISABLE!
# If you change this first entry you will need to make sure that the
# database superuser can access the database using some other method.
# Noninteractive access to all databases is required during automatic
# maintenance (custom daily cronjobs, replication, and similar tasks).
#
# Database administrative login by Unix domain socket
local   all             postgres                                trust

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5
```
You can save the file by using `CTRL+O` (write out) and after that `CTRL+X` (exit).

Restart PostgreSQL to make the change effective.
```shell
sudo service postgresql restart
```
PostgreSQL is now ready. To test the configuration, try to connect to the `geonode` database as `geonode` role:

```shell
psql -U postgres geonode
# This should not ask for any password
# use \q to quit

psql -U geonode geonode
# This should ask for the password geonode
# use \q to quit

# Repeat the test with geonode_data DB
# use \q to quit
psql -U postgres geonode_data
psql -U geonode geonode_data
```

## reset dev environment and sync it again

Because we changed the backend we will have to reset our dev setup and migrate it and install the fixtures again.
First lets reset the current dev setup. To accomplish this we must use the terminal in PyCharm with activated virtual
environment. See:

![resetting paver](img/paver_reset.png)

```shell
paver reset
```

We will use `.env` files to configure our environmental variables and development setup. I have created two files in this
repository you can use. You should the user and password and database names like you used before to create the databases.
If you followed my instructions they should be:

`postgreSQL.env`
```dotenv
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
GEONODE_DATABASE=geonode
GEONODE_DATABASE_PASSWORD=geonode
GEONODE_GEODATABASE=geonode_data
GEONODE_GEODATABASE_PASSWORD=geonode_data
GEONODE_DATABASE_SCHEMA=public
GEONODE_GEODATABASE_SCHEMA=public
DATABASE_URL=postgis://geonode:geonode@localhost:5432/geonode
GEODATABASE_URL=postgis://geonode:geonode@localhost:5432/geonode_data
GEONODE_DB_CONN_MAX_AGE=0
GEONODE_DB_CONN_TOUT=5
DEFAULT_BACKEND_DATASTORE=datastore
```

`dev.env`
```dotenv
DEBUG=True
DJANGO_SETTINGS_MODULE=geonode.settings
```
You can see a list of possible environmental variables in the `.env` file provided inside the 
GeoNode repo [here](https://github.com/GeoNode/geonode/blob/master/.env) or take a look at the 
[settings.py file](https://github.com/GeoNode/geonode/blob/master/geonode/settings.py). All the 
`os.getenv()` are read by the environmental variables set inside the host.

Since we deleted our development sqlite database we need to migrate the needed database tables to our PostgreSQL 
database. Since we will use the `paver sync` command we have to set our environmental variables inside the `.env` files 
on the host. This can be achieved by parsing the files like:

```shell
# load the dev.env variables
set -o allexport; source dev.env; set +o allexport
# load the postgreSQL.env variables
set -o allexport; source postgreSQL.env; set +o allexport

# sync the postgreSQL databases
paver sync

# start geoserver
paver start_geoserver
```

## reading .env files in PyCharm as run configurations

With a completly migrated database and GeoServer running we can focus on getting our django debug configuration to run 
with our `.env` files. You can use them inside the run configuration menu of PyCharm like this:

switch to the EnvFile section
![run config menu](img/Debug_Config_EnvFiles.png)

enable EnvFile parsing and add files:
![run config menu](img/DebugConfig_addEnvFiles.png)

Add the envFiles provided by this repo:
![run config menu](img/DebugConfig_addedEnvFilesDone.png)

This will enable the debug run configuration to read the environmental variables and use them on startup. Time to test
the new setup. Go ahead, run the debug view and upload some files. You already know how to do this.

congratulations! You have a running development environment with a proper PostgreSQL backend to work with and know how to
use `.env` files to configure your setup. You eg. could enable mailing or the creation of layers to debug those features.