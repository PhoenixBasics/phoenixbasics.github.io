---
layout: page
title: Create a new project
---


To create a new project, type:

```
mix phx.new [project name]
```

Example:

```
mix phx.new fawkes
```

Running that command will generate the Phoenix application for you. After the file creation, it will ask you to fetch and install dependencies. Type `y`.

(if you're following along with the https://github.com/PhoenixBasics/fawkes - you'll want to run `mix phx.new` in the folder where you git cloned the repo... not in the project folder itself.)

```
Fetch and install dependencies? [Yn] y
```

By saying yes, it these commands will be run for you:

This commands downloads your dependencies:

```
mix deps.get
```

Phoenix uses Brunch.io for asset management by default. Brunch.io’s dependencies are installed via the node package manager, not mix. SoT this command will install your node dependencies:

```
cd assets && npm install && node node_modules/brunch/bin/brunch build
```

This compiles the project:

```
mix deps.compile
```

Note that Phoenix will auto-reload your code for you so you don't have to run compile every time you make a file change.

## Running the server
After the application installed your dependencies, it tells you what to do next.

1. Change into your project directory

  ```
  cd fawkes
  ```

2. Open the application in your favorite editor
3. Open the file `config/dev.exs` and Ensure your username and password for Postgres is correct.
4. Ecto allows our Phoenix application to communicate with a data store -- PostgreSQL in this case. Create your database from the commandline by running:

  ```
  mix ecto.create
  ```

5. Start your server

  ```
  mix phx.server
  ```

6. Open [http://localhost:4000](http://localhost:4000) in your favorite browser.

## Congratulations, you got a server!!!

<img src="https://media.giphy.com/media/10Fqkgb4tQVtOo/giphy.gif">


### Prettifying the page
Download these assets for styling and the about page:

```
curl -o assets/css/zapp.css https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/assets/css/zapp.css && curl -o lib/fawkes_web/templates/layout/app.html.eex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/templates/layout/app.html.eex && curl -o assets/static/images/elixirconf.png  https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/assets/static/images/elixirconf.png && curl -o assets/static/images/hyatt.png  https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/assets/static/images/hyatt.png

```
