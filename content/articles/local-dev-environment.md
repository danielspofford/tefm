# Local dev environment

## Docker Compose

Docker Compose can be leveraged to handle services you need to run locally in a
consistent manner across developers.

A `docker-compose.yml` file in the project root with the follow contents:

```yaml
version: '2'
services:
  db:
    image: postgres:10.5
    ports:
     - "5432:5432"
```

Can be leveraged via:

```shell
$ docker-compose up -d
$ mix ecto.create
```
