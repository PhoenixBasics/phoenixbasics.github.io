---
layout: page
title: Create a new project
---

Let's clone the project.

```
git clone https://github.com/PhoenixBasics/fawkes
```

Don't navigate into the project. From where we clone the repo, let's create a new project named `fawkes`, type:

```
mix phx.new fawkes
```

That command will generate the Phoenix application for us. After the file creation, it will ask us to fetch and install dependencies. Type `y`.

```
Fetch and install dependencies? [Yn] y
```

By saying yes, these commands will be run for us:

This commands downloads our dependencies:

```
mix deps.get
```

Phoenix uses Brunch.io for asset management by default. Brunch.io’s dependencies are installed via the node package manager, not mix. This command will install our node dependencies:

```
cd assets && npm install && node node_modules/brunch/bin/brunch build
```

This compiles the project:

```
mix deps.compile
```

Note that Phoenix will auto-reload our code for us so we don't have to run compile every time we make a file change.

## Running the server
After the application installed our dependencies, it tells us what to do next.

1. Change into our project directory

  ```
  cd fawkes
  ```

2. Open the application in our editor
3. Open the file `config/dev.exs` and ensure the username and password for Postgres is correct.
4. Ecto allows our Phoenix application to communicate with a data store -- PostgreSQL in this case. Create our database from the command line by running:

  ```
  mix ecto.create
  ```

5. Start our server

  ```
  mix phx.server
  ```

6. Open [http://localhost:4000](http://localhost:4000) in a browser.

## Congratulations, we got a server!!!

<img src="https://media.giphy.com/media/10Fqkgb4tQVtOo/giphy.gif">


### Prettifying the page
Merge in the assets branch to get the assets for our application.

```
git merge assets
```
