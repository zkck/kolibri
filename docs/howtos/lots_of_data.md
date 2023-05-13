# Simulating a large data transfer with a PostgreSQL backend

This document is for describing how to set up Kolibri in order to simulate performance bottlenecks during the data transfer syncing in Morango. We 

## First server

We will first setup a base server which we will populate with dummy data through a management command.


### Frontend & backend

First step is to start the server with a PostgreSQL backend. For this we can have a look at the Makefile at the root of the Kolibri source code, which has the following lines:

```
%-with-postgres:
	@echo -e "\e[33mWARNING: for testing purposes only; postgresql database backend is ephemeral\e[0m"
	@echo -e "\e[36mINFO: run 'docker-compose -v' to remove the database volume\e[0m"
	export KOLIBRI_DATABASE_ENGINE=postgres; \
	export KOLIBRI_DATABASE_NAME=default; \
	export KOLIBRI_DATABASE_USER=postgres; \
	export KOLIBRI_DATABASE_PASSWORD=postgres; \
	export KOLIBRI_DATABASE_HOST=127.0.0.1; \
	export KOLIBRI_DATABASE_PORT=15432; \
	set -ex; \
	function _on_interrupt() { docker-compose down; }; \
	trap _on_interrupt SIGINT SIGTERM SIGKILL ERR; \
	docker-compose up --detach; \
	until docker-compose logs --tail=1 postgres | grep -q "database system is ready to accept connections"; do \
		echo "$(date) - waiting for postgres..."; \
		sleep 1; \
	done; \
	$(MAKE) -e $(subst -with-postgres,,$@); \
	docker-compose down -v
```

And we can also have a look at the `python-devserver` `yarn` command mentioned in the documentation, in `package.json`:

```
    ...
    "python-devserver": "kolibri start --debug --foreground --port=8000 --settings=kolibri.deployment.default.settings.dev",
    ...
```

To then build the frontend, we can run `yarn build`.

### PostgreSQL

We can then bring PostgreSQL instance up with docker-compose, as the Kolibri codebase has a `docker-compose.yml` at its root:

```bash
docker compose up
```

Then, after setting up your Python virtual environment according to the documentation at, you can then start the Kolibri server with the following:

```bash
export KOLIBRI_DATABASE_ENGINE=postgres
export KOLIBRI_DATABASE_NAME=default
export KOLIBRI_DATABASE_USER=postgres
export KOLIBRI_DATABASE_PASSWORD=postgres
export KOLIBRI_DATABASE_HOST=127.0.0.1
export KOLIBRI_DATABASE_PORT=15432
kolibri start --debug --foreground --port=8000 --settings=kolibri.deployment.default.settings.dev
```

### Populating

For generating data, we will be using the `generateuserdata` management command, whose source can be found at TODO. For this to generate data in the right location, we need to again set the above environment variables pointing to the right database type and location:

```
export KOLIBRI_DATABASE_ENGINE=postgres
export KOLIBRI_DATABASE_NAME=default
export KOLIBRI_DATABASE_USER=postgres
export KOLIBRI_DATABASE_PASSWORD=postgres
export KOLIBRI_DATABASE_HOST=127.0.0.1
export KOLIBRI_DATABASE_PORT=15432
kolibri manage generateuserdata
```

This on its own will output some logs about users being inserted, but will include the following log line:

```
...
No channels found, cannot add channel activities for learners.
...
```

This limits us in how much data we can generate in order to perform an effective load test. We can create a channel by logging into our Kolibri in our web browser, and adding a channel from Kolibri Studio.

So we can now import a resource from a channel, and re-run the generateuserdata command. 

## Second server

We then need to run a second server, also with a PostgreSQL backend. This can be done by changing the exposed port (changed ot 9000), and the database to which Kolibri connects to (changed to `default2`):

```
export KOLIBRI_DATABASE_ENGINE=postgres
export KOLIBRI_DATABASE_NAME=default2
export KOLIBRI_DATABASE_USER=postgres
export KOLIBRI_DATABASE_PASSWORD=postgres
export KOLIBRI_DATABASE_HOST=127.0.0.1
export KOLIBRI_DATABASE_PORT=15432
export KOLIBRI_HOME=$PWD/second-home
kolibri start --debug --foreground --port=9000 --settings=kolibri.deployment.default.settings.dev
```

## Triggering a sync



## Profiling
