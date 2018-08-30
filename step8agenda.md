---
layout: page
title: API - Add talks to agenda
---

We need a way for user to add a talk to their agenda. Run this command to generate a user talk relationship.

```
mix phx.gen.json Agenda UserTalk users_talks user_id:references:users talk_id:references:talks
```

Open up `lib/fawkes/agenda/user_talk.ex`. Add `:user_id` and `:talk_id` to the cast so your changeset will look like this:

```
def changeset(user_talk, attrs) do
  user_talk
  |> cast(attrs, [:user_id, :talk_id])
  |> validate_required([])
end
```

In the `router`, add an api scope for `user_talk`:

```
scope "/api", FawkesWeb do
  pipe_through [:api]

  resources "/users_talks", UserTalkController, except: [:new, :edit]
end
```

Now you're ready to save! Go to [http://localhost:4000/schedule_slots](http://localhost:4000/schedule_slots), click add to agenda on a few slot.

### Showing my agenda

Let's get our agendas. In your `router`, below `resources "/speakers", SpeakerController`, add a path to retrieve your agenda

```
get "/agendas", AgendaController, :index
```

When the user hits this endpoint, we will retrieve all the talks for the current user. Let's create a new controller called `lib/fawkes_web/controller/agenda_controller.ex`. Add this code:

```
defmodule FawkesWeb.AgendaController do
  use FawkesWeb, :controller

  alias Fawkes.Agenda

  def index(conn, _params) do
    talks = Agenda.list_talk_for_user(conn.assigns.current_user)
    render(conn, "index.html", talks: talks)
  end
end
```

Open `lib/fawkes/agenda/agenda.ex`, add in this function to retrieve the talks:

```
alias Fawkes.Schedule.Talk

def list_talk_for_user(%{id: id}) do
  Talk
  |> preload([:slot, :speakers, :categories, :audience, :location])
  |> join(:inner, [talk], user_talk in UserTalk, user_talk.talk_id == talk.id)
  |> Repo.all()
end
```

Now that we retrieved the talk, let's display it. Create a new file called `lib/fawkes_web/views/agenda_view.ex`. Add this code to render the view:

```
defmodule FawkesWeb.AgendaView do
  use FawkesWeb, :view
end
```

Now let's get the HTML for the view. Create a new folder called `lib/fawkes_web/templates/agenda`. Create a new file called `lib/fawkes_web/templates/agenda/index.html.eex`. Add this code to display the data:

```
<div class="talk-index">
  <div class="section-white">
    <div class="container">
      <div class="d-flex justify-content-center align-items-center dates">
        <i class="fas fa-arrow-left"></i>
        <%= for {day_number, day_name} <- [{4, "Tue"}, {5, "Wed"}, {6, "Thu"}, {7, "Fri"}] do %>
          <div class="card">
            <div class="card-body">
              <div class="dates__name smalltext"><%= day_name %></div>
              <div class="dates__number"><%= day_number %></div>
            </div>
          </div>
        <% end %>
        <i class="fas fa-arrow-right"></i>
      </div>
      <div class="d-flex justify-content-center talk-index__tab">
        <a href="/schedule_slots" class="talk-tab__link"><h4 class="subheader">Full Schedule</h4></a>
        <a href="/agenda" class="talk-tab__link talk-tab__link-active"><h4 class="subheader">My Agenda</h4></a>
      </div>
    </div>
  </div>

  <div class="container schedule">
    <%= for talk <- @talks do %>
      <div class="card">
        <div class="card-body">
          <div class="float-right schedule__talk-link"><i class="fas fa-angle-right"></i></div>
          <h4 class="subheader"><%= Timex.format!(talk.slot.start_time, "%-I:%M %p", :strftime)%> - <%= Timex.format!(talk.slot.end_time, "%-I:%M %p", :strftime) %></h4>
          <a href="/talks/<%= talk.id %>"><h3 class="schedule__talk-title"><%= talk.title %></h3></a>

          <div class="schedule__details"><%= FawkesWeb.SharedView.full_name(talk.speakers) %> &nbsp; | &nbsp; <%= talk.location.name %></div>
          <p class="icons">
            <button class="btn talk__audience"><%= talk.audience.name %></button>
            <%= for category <- talk.categories do %>
                <%= if category.icon_url do %>
                  <img src="<%= category.icon_url %>">
                <% else %>
                <button class="btn talk__category">
                  <%= category.name %>
                </button>
                <% end %>
            <% end %>
          </p>
          <div class="add-to-schedule action">
            <i class="far fa-calendar-check"></i>
            ADD TO AGENDA
          </div>
        </div>
      </div>
    <% end %>
  </div>
</div>
```

Go to [http://localhost:4000/agendas](http://localhost:4000/agendas). Note the talks you added are there.

### Exercise: View other user agenda
1. Add a new endpoint `/agendas/:user_plug`
2. Add a new function in the `lib/fawkes_web/controller/agenda_controller.ex`
  a. First find the user with the given slug
  b. Find the user agenda. You should be able to reuse `Agenda.list_talk_for_user(conn.assigns.current_user)`
3. Since you're only displaying the talk, you can reuse the index template `render(conn, "index.html", talks: talks)`
