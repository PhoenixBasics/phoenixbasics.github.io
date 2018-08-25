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

```
Fetch and install dependencies? [Yn] y
```

By saying yes, it ran these commands for you:

This commands downloads your dependencies:

```
mix deps.get
```

Phoenix uses Brunch.io for asset management by default. Brunch.io’s dependencies are installed via the node package manager, not mix. This command install your node dependencies:

```
cd assets && npm install && node node_modules/brunch/bin/brunch build
```

This compile the project:

```
mix deps.compile
```

Note that Phoenix auto reload for you so you don't have to run compile every time you make a file change.

## Running the server
After the application installed your dependencies, it tells you what to do next.

1. Change into your project directory

  ```
  cd fawkes
  ```

2. Open the application in your favorite editor
3. Open the file `config/dev.exs`. Ensure your username and password for Postgres is correct.
2. Ecto allows our Phoenix application to communicate with a data store, such as PostgreSQL or MongoDB. Create your database by running:

  ```
  mix ecto.create
  ```

4. Start your server

  ```
  mix phx.server
  ```

5. Open [http://localhost:4000](http://localhost:4000) in your favorite browser.

## Congratulations, you got a server!!!

<img src="https://media.giphy.com/media/10Fqkgb4tQVtOo/giphy.gif">


### Prettifying the page
Download these assets for styling and the about page:

```
curl -o assets/css/zapp.css https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/assets/css/zapp.css && curl -o lib/fawkes_web/templates/layout/app.html.eex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/templates/layout/app.html.eex && curl -o assets/static/images/elixirconf.png  https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/assets/static/images/elixirconf.png && curl -o assets/static/images/hyatt.png  https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/assets/static/images/hyatt.png

```
