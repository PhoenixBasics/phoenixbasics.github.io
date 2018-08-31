---
layout: page
title: Populate the database
---
(Use git checkout `5.seed` to catch up with the class)

Now that we have our modules, let's connect them all together?

Run this command to get the seed files:

```
git merge seeds
```

Add this line to `priv/repo/seeds.exs` run the seed file:

```
Fawkes.Schedule.Seed.perform()
```

Let's seed our database:

```
mix run priv/repo/seeds.exs
```

### Querying for data
(Use git checkout `5a.querydata` to catch up with the class)

When we load the schedule, we want to load the talk, slot, location, and category. Let's open `lib/fawkes/schedule/schedule.ex`. First thing we want to do add the ability to query. So import in `Ecto.Query` on line 4.

```
import Ecto.Query
```

We're going to load all the talks through the slots. Go to the function `list_schedule_slots` (maybe line 126). We want to say load all the event and talks. For all the talks, list the slot, speakers, categories, audience, and location.

```
Slot
|> preload([:event, talks: [:slot, :speakers, :categories, :audience, :location]])
|> order_by(asc: :start_time)
|> Repo.all()
|> Enum.group_by(&(Timex.beginning_of_day(&1.start_time)))
```

Now let's update our layout by merging the the schedule_layouts branch.

```
git merge schedule_layouts
```

Restart your server. Press `Cmd` + `c` to quit. Then run:

```
mix phx.server
```


### Exercise: Load speakers
(Use git checkout `5b.load_speaker` to catch up with the class)

1. In the `lib/fawkes/schedule/schedule.ex`, find the function `list_profiles`. Change it to `list_speakers`.
2. It should preload `talk`
3. Open `lib/fawkes_web/controllers/speaker_controllers.ex`, update the `index` function to reference speakers instead of profile. The code should look like this:

  ```
    def index(conn, _params) do
      speakers = Schedule.list_speakers()
      render(conn, "index.html", speakers: speakers)
    end
  ```

3. Go to [http://localhost:4000/speakers](http://localhost:4000/speakers), make sure the speaker loads

### Exercise #2: Show talk

Lab Add path for talk

1. In `lib/fawkes/schedule/schedule.ex`, find the function `get_talk`. Preload slot, speakers, categories, audience, location
2. Add a template for the view. Here's the HTML for it.

```
<div class="container talk">
  <div class="card section">
    <p class="subtext"><%= Timex.format!(@talk.slot.start_time, "%-I:%M %p", :strftime)%> - <%= Timex.format!(@talk.slot.end_time, "%-I:%M %p", :strftime) %></p>
    <h2><%= @talk.title %></h2>
    <p class="smalltext"><strong>Location</strong>: <%= @talk.location.name %></p>
    <p class="icons">
      <button class="btn talk__audience"><%= @talk.audience.name %></button>
      <%= for category <- @talk.categories do %>
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

  <hr/>
  <h4 class="subheader">Speakers/Trainers</h4>
  <%= for speaker <- @talk.speakers do %>
    <div class="card speaker">
      <div class="row card-body">
        <div class="col-sm-3">
          <img src="<%= speaker.image_url %>" class="rounded-circle">
        </div>
        <div class="col-sm-7 speaker__info">
          <a href="/speakers/<%= speaker.id %>"<h4><%= FawkesWeb.SharedView.full_name(speaker) %></h4></a>
          <div class="speaker__handle">
            <i class="fab fa-github"></i> @<%= speaker.github %>
          </div>
          <div class="speaker__handle">
            <i class="fab fa-twitter"></i> @<%= speaker.twitter %>
          </div>
        </div>
        <div class="col-sm-1 speaker__link">
          <a href="/speakers/<%= speaker.id%>"><i class="fas fa-chevron-right"></i></a>
        </div>
      </div>
    </div>
  <% end %>
  <hr />
  <h4 class="subheader">About the event</h4>
  <p><%= raw(@talk.description) %></p>

  <hr />
  <h4 class="subheader">Checked in attendees</h4>
  <div class="row attendee">
    <%= for _ <- (1..10) do %>
      <div class="card col-sm-3">
        <div class="card-body">
          <img src="https://elixirconf.com/2018/images/speakers/boyd-multerer.jpg" class="rounded-circle">
          <i class="fas fa-check-circle attendee__check"></i>
          <h4>Sandra Bullock</h4>
          <div class="speaker__handle">
            <i class="fab fa-github"></i> @TODO-GIthubhandle
          </div>
          <div class="speaker__handle">
            <i class="fab fa-twitter"></i> @TODO-twitterhandle
          </div>
        </div>
      </div>
    <% end %>
  </div>
</div>
```

Go to [http://localhost:4000/talks/1](http://localhost:4000/talks/1)
It should work.
