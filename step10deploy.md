---
layout: page
title: Deploy
---

### Configs

1. Open up `config/prod.exs`, add a comma to `  cache_static_manifest: "priv/static/cache_manifest.json",`. Then add this below it:

```
secret_key_base: Map.fetch!(System.get_env(), "SECRET_KEY_BASE")
```

Scroll to the bottom of the file, remove `import_config "prod.secret.exs"`. Add this database setting to the file:

```
config :fawkes, Fawkes.Repo,
  adapter: Ecto.Adapters.Postgres,
  url: System.get_env("DATABASE_URL"),
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
  ssl: true
```

Your `config/prod.exs` file should look like this (without the comments):

```
use Mix.Config

config :fawkes, FawkesWeb.Endpoint,
  load_from_system_env: true,
  url: [host: "example.com", port: 80],
  cache_static_manifest: "priv/static/cache_manifest.json",
  secret_key_base: Map.fetch!(System.get_env(), "SECRET_KEY_BASE")

config :logger, level: :info

config :fawkes,
   FawkesWeb.Guardian.Tokenizer,
   issuer: "fawkes",
   secret_key: System.get_env("GUARDIAN_KEY")

config :fawkes, Fawkes.Repo,
  adapter: Ecto.Adapters.Postgres,
  url: System.get_env("DATABASE_URL"),
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
  ssl: true
```

### Deploy

1. Create a new file in the root application called `deploy.sh`
2. Add this content to the file

  ```
  heroku create --buildpack "https://github.com/HashNuke/heroku-buildpack-elixir.git"
  heroku buildpacks:add https://github.com/gjaldon/heroku-buildpack-phoenix-static.git
  touch Procfile
  echo "web: MIX_ENV=prod mix phx.server" > Procfile
  heroku addons:create heroku-postgresql:hobby-dev
  heroku config:set POOL_SIZE=18
  heroku config:set SECRET_KEY_BASE=""
  heroku config:set GUARDIAN_KEY=""

  heroku config:set S3_BUCKET="aaa"
  heroku config:set AWS_ACCESS_KEY="aaa"
  heroku config:set AWS_SECRET_KEY="aaa"

  git push heroku master

  heroku run "POOL_SIZE=2 mix ecto.migrate"
  heroku run "POOL_SIZE=2 mix run priv/repo/seeds.exs"
  ```

3. Run `mix phx.gen.secret` and set the SECRET_KEY_BASE to the generated string
4. Run `mix phx.gen.secret` and set the GUARDIAN_KEY to the generated string
5. Add in your S3 bucket, access key, and secret key
6. If you're on a branch, change `git push heroku master` to `git push heroku branchname:master`
7. Run `chmod u+x deploy.sh` to make the file executable
8. Run `./deploy.sh`
