---
layout: page
title: Add a feature
---


Now we know how to add a route, let's add a new feature. We're going to add categories to tag what type of talk it is.

### Add a model

Let's use Phoenix to generate a model called Category.


```
mix phx.gen.schema Schedule.Category categories slug:string:unique name:string icon_url:string

```

DON'T run the migration yet.

Running this command will create two files, a migration file (`priv/repo/migrations/20180803152910_create_categories.exs`) and a model file (`lib/fawkes/schedule/category.ex`). The migration file uses the timestamp to know which file to migrate first. https://hexdocs.pm/ecto/Ecto.Schema.html

Our project uses Ecto to talk to the database. [Ecto](https://hexdocs.pm/ecto/Ecto.html) is a library that handles form of data validation and persistence. It consist of 4 parts:
- Ecto.Repo - repositories are wrappers around the data store. Via the repository, we can create, update, destroy and query existing entries. A repository needs an adapter and credentials to communicate to the database
- Ecto.Schema - schemas are used to map any data source into an Elixir struct
- Ecto.Changeset - changesets provide a way for developers to filter and cast external parameters, as well as a mechanism to track and validate changes before they are applied to your data
- Ecto.Query - written in Elixir syntax, queries are used to retrieve information from a given repository. Queries in Ecto are secure, avoiding common problems like SQL Injection, while still being composable, allowing developers to build queries piece by piece instead of all at once

You will learn more about as we continue to build this application.

Let's add an index for the category name.

```
create index("categories", :name, unique: true)
```

By default, Ecto will make every field required. But we don't want the icon_url to be required because not all of them have an image associated with it. Let's open up `lib/fawkes/schedule/category.ex` and remove `icon_url` from the changeset validate_required function (line 18). So it would only require name and slug.

```
    |> validate_required([:slug, :name])
```

Now run this command to create the table in the database:

```
mix ecto.migrate
```

### Add a Context

Create a new file called `lib/fawkes/schedule/schedule.ex`. Add the following code to the file:

```
defmodule Fawkes.Schedule do
  alias Fawkes.Repo
  alias Fawkes.Schedule.Category

  def all_category do
    Repo.all(Category)
  end

end
```

This will call the repository to get all the category.

### Add a Controller

Now let's add a controller to handle the request. Create a new file called `lib/fawkes_web/controllers/category_controller.ex`. Add the following code to the file:

```
defmodule FawkesWeb.CategoryController do
  use FawkesWeb, :controller
  alias Fawkes.Schedule

  def index(conn, _params) do
    categories = Schedule.all_category()
    render(conn, "index.html", categories: categories)
  end

end

```

Let's add a presenter by creating a file called `lib/fawkes_web/views/category_view.ex`. Add the following code to the view:

```
defmodule FawkesWeb.CategoryView do
  use FawkesWeb, :view

end
```

Create a new folder called `category` in `lib/fawkes_web/templates`. Add a new file called `lib/fawkes_web/templates/index.html.eex`. Add the following code to display the category.

```
<h2>Categories</h2>
<a href="/categories/new">New Category</a>

<table class="table">
  <thead>
    <tr>
      <th>Name</th>
      <th>Slug</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
  <%= for category <- @categories do %>
      <tr>
        <td><%= category.name %></td>
        <td><%= category.slug %></td>
        <td><a href="/categories/<%= category.id %>">Show</a></td>
      </tr>
  <% end %>
  </tbody>
</table>
```

Open `lib/fawkes_web/router.ex`. On line 21, after the `get "/", PageController, :index` line, add the following code:

```
 get "/categories", CategoryController, :index
```

Now go to [http://localhost:4000/categories](http://localhost:4000/categories)

Woohoo! We have a page for the category. Since we don't have any category, our page is empty. Let's add a page to create a category.

### Add a category form

First let's create a form to add a category. Create a new file called `lib/fawkes_web/templates/category/new.html.eex`. Add the following code to the file:

```
<h2>New Category</h2>
<%= form_for @changeset, category_path(@conn, :create), fn f -> %>
  <%= if @changeset.action do %>
    <div class="alert alert-danger">
      <p>Oops, something went wrong! Please check the errors below.</p>
    </div>
  <% end %>

  <div class="form-group">
    <%= label f, :name, class: "control-label" %>
    <%= text_input f, :name, class: "form-control" %>
    <%= error_tag f, :name %>
  </div>

  <div class="form-group">
    <%= label f, :slug, class: "control-label" %>
    <%= text_input f, :slug, class: "form-control" %>
    <%= error_tag f, :slug %>
  </div>

  <div class="form-group">
    <%= submit "Submit", class: "btn btn-primary" %>
  </div>
<% end %>
```

In `lib/fawkes_web/router.ex`, add a new line on line 21, before the category index. Add a new path to get the form:

```
get "/categories/new", CategoryController, :new
```

Open `lib/fawkes_web/controllers/category_controller.ex`, add a new function to show a form for a new category:

```
def new(conn, _params) do
  render(conn, "new.html", changeset: Schedule.category_changeset())
end
```

Now open `lib/fawkes/schedule/schedule.ex`. Add this method to get a new changeset.

```
def category_changeset(changeset \\ %Category{}) do
  Category.changeset(changeset, %{})
end
```

Ok, now go to http://localhost:4000/categories/new. Notice it gives you an error because we didn't define the path to post the data.

### Create a category

We need to add a function to save the data to the database. Open `lib/fawkes/schedule/schedule.ex` and add this code:

```
def create_category(attrs \\ %{}) do
  %Category{}
  |> Category.changeset(attrs)
  |> Repo.insert()
end
```

In your `CategoryController` add this function to save it in the database.

```
def create(conn, %{"category" => params}) do
  case Schedule.create_category(params) do
    {:ok, _category} ->
      conn
      |> put_flash(:info, "Category created successfully.")
      |> redirect(to: category_path(conn, :index))
    {:error, %Ecto.Changeset{} = changeset} ->
      render(conn, "new.html", changeset: changeset)
  end
end
```

In your router (`lib/fawkes_web/router.ex`), add a new path post call to create the category. Add this after the `get "/categories/new", CategoryController, :new`

```
post "/categories", CategoryController, :create
```

Go to http://localhost:4000/categories/new, add a new category and hit submit. Congrats. It is saving in the database.


### Show
Exercise #1: Implement the show category

When a user sends GET request to `http://localhost:4000/categories/1`, the application will displays the category's name, slug, icon_url.
Hint: To get a category from a repo: `Repo.get!(Category, id)`

### Resources:
- [https://hexdocs.pm/ecto/Ecto.Repo.html](https://hexdocs.pm/ecto/Ecto.Repo.html)
- [https://hexdocs.pm/phoenix/Phoenix.Controller.html](https://hexdocs.pm/phoenix/Phoenix.Controller.html)

### Delete
Optional Exercise: Implement the delete feature.

When a user sends DELETE request to `http://localhost:4000/categories/1`, the application will delete the category with ID 1 and redirect the user to `/categories`.

Hint: To delete a category from a repo `Repo.delete(category)`. To redirect to category index `redirect(conn, to: category_path(conn, :index))`
