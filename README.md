## Install docker

```sh
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker ${USER}
docker ps
```

## Build docker image
```sh
docker build -t local-synapse -f docker/Dockerfile .
```

## Install postgres

```sh
sudo apt install -y postgresql-common && sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh	
sudo apt install postgresql-17 postgresql-contrib-17 -y	
sudo systemctl enable postgresql	

sudo -u postgres psql	
	CREATE USER synapse WITH PASSWORD 'Abc@123456';
	ALTER USER synapse WITH SUPERUSER;
	\q

sudo nano /etc/postgresql/17/main/pg_hba.conf	
  host all all 0.0.0.0/0 md5
  host all all ::1/128 md5

sudo nano /etc/postgresql/17/main/postgresql.conf	
  listen_addresses = '*'

sudo systemctl restart postgresql	
```

## Create DB
```sh
createdb --encoding=UTF8 --locale=C --template=template0 --owner=synapse -U synapse -h localhost synapse-db
```

## Generating a configuration file

The first step is to generate a valid config file. To do this, you can run the
image with the `generate` command line option.

You will need to specify values for the `SYNAPSE_SERVER_NAME` and
`SYNAPSE_REPORT_STATS` environment variable, and mount a docker volume to store
the configuration on. For example:

```
docker run -it --rm \
    --mount type=volume,src=synapse-data,dst=/data \
    -e SYNAPSE_SERVER_NAME=my.matrix.host \
    -e SYNAPSE_REPORT_STATS=yes \
    local-synapse:latest generate
```

## Update configuration

```
sudo nano /var/lib/docker/volumes/synapse-data/_data/homeserver.yaml
  database:
    name: psycopg2
    args:
      user: synapse
      password: Abc@123456
      dbname: synapse-db
      host: 172.17.0.1
      cp_min: 5
      cp_max: 10
```

## Running synapse

Once you have a valid configuration file, you can start synapse as follows:

```
docker run -d --name synapse \
    --mount type=volume,src=synapse-data,dst=/data \
    -p 8008:8008 \
    local-synapse:latest
```

(assuming 8008 is the port Synapse is configured to listen on for http traffic.)

You can then check that it has started correctly with:

```
docker logs synapse
```

## Generating an (admin) user

After synapse is running, you may wish to create a user via `register_new_matrix_user`.

This requires a `registration_shared_secret` to be set in your config file. Synapse
must be restarted to pick up this change.

You can then call the script:

```
docker exec -it synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml --help
```


```
docker exec -it synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml -u admin -p Abc@123456 -a

```


