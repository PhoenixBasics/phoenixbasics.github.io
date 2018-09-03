---
layout: page
title: Add a feature - Answer
---

## Show Category

In `schedule.ex`, add

```
def get_category!(id) do
  Repo.get!(Category, id)
end
```

In `category_controller.ex`, add

```
def show(conn, %{"id" => id}) do
  category = Schedule.get_category!(id)
  render(conn, "show.html", category: category)
end
```

Create a new file `lib/fawkes_web/templates/category/show.html.eex` and add this code:

```
<h1>Category</h1>
<p>Name: <%= @category.name %></p>
<p>Slug: <%= @category.slug %></p>
<p>Icon URL: <%= @category.icon_url %></p>
```

In the `router.ex` file, add this path below new category:

```
get "/categories/:id", CategoryController, :show
```

## Delete Category

(Use `git checkout 3f.delete_category` to catch up with the class)

In the `lib/fawkes_web/templates/category/index.html.eex`, add this next to show:

```
<span><%= link "Delete", to: category_path(@conn, :delete, category), method: :delete, data: [confirm: "Are you sure?"], class: "btn btn-danger btn-xs" %></span>
```

In the `router.ex`, add this code below show category

```
delete "/category/:id", CategoryController, :delete
```

In the `category_controller`, add this:

```
def delete(conn, %{"id" => id}) do
  {:ok, _category} = Schedule.delete_category(id)

  conn
  |> put_flash(:info, "Category deleted.")
  |> redirect(to: category_path(conn, :index))  end
end
```

In the schedule, add this to delete category:

```
def delete_category(category) do
  id
  |> get_category!()
  |> Repo.delete()
end
```
