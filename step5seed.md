---
layout: page
title: Populate the database
---
(Use git checkout `5.seed` to catch up with the class)

Now that we have our modules, let's connect them all together?

Run this command to get the seed files:

```
git merge origin/seeds
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

Now, we need to add a shared view to help with formatting the layout. Create a filed called `lib/fawkes_web/views/shared_view.ex`. Add this code:

```
defmodule FawkesWeb.SharedView do
  use FawkesWeb, :view

  def full_name(people) when is_list(people) do
    people
    |> Enum.map(&full_name/1)
    |> Enum.join(" && ")
  end

  def full_name(person) do
    person.first <> " " <> person.last
  end
end
```

Finally, let's update our layout. Open `lib/fawkes_web/templates/slot/index.html.eex`, and replace the code with this:

```
<div class="talk-index">
  <div class="section-white">
    <div class="container">
      <div class="d-flex justify-content-center align-items-center dates">
        <i class="fas fa-arrow-left"></i>
        <%= for date <- conference_dates() do %>
          <div class="card">
            <div class="card-body">
              <a href="/schedule_slots?date=<%= Timex.format!(date, "%Y-%m-%d", :strftime)%>">
                <div class="dates__name smalltext"><%= Timex.format!(date, "%a", :strftime) %></div>
                <div class="dates__number"><%= Timex.format!(date, "%e", :strftime) %></div>
              </a>
            </div>
          </div>
        <% end %>
        <i class="fas fa-arrow-right"></i>
      </div>
      <div class="d-flex justify-content-center talk-index__tab">
        <a href="/schedule_slots" class="talk-tab__link talk-tab__link-active"><h4 class="subheader">Full Schedule</h4></a>
        <%= if @conn.assigns[:current_user] do %>
          <a href="/agenda" class="talk-tab__link"><h4 class="subheader">My Agenda</h4></a>
        <% end %>
      </div>
    </div>
  </div>

  <div class="container schedule">
    <p class="alert alert-info" role="alert"></p>
    <p class="alert alert-danger" role="alert"></p>
    <%= for {date, slots} <- @schedule_slots do %>
      <h2 class="schedule__date"><%= Timex.format!(date, "%A  %m/%d/%y", :strftime)%></h2>
      <%= for slot <- slots do %>
        <div class="card">
          <div class="card-header">
            <h4 class="subheader schedule__time">
              <%= Timex.format!(slot.start_time, "%-I:%M %p", :strftime)%> - <%= Timex.format!(slot.end_time, "%-I:%M %p", :strftime) %>
            </h4>
          </div>
          <div class="card-body">
            <%= if slot.event do %>
              <h3 class="gray"><%= slot.event.name %></h3>
            <% else %>
              <%= for talk <- slot.talks do %>
                <div class="schedule__talk">
                  <div class="float-right schedule__talk-link">
                    <i class="fas fa-angle-right"></i>
                  </div>
                  <a href="/talks/<%= talk.id %>">
                    <h3 class="schedule__talk-title">
                      <%= talk.title %>
                    </h3>
                  </a>

                  <div class="schedule__details">
                    <%= FawkesWeb.SharedView.full_name(talk.speakers) %> &nbsp; | &nbsp; <%= talk.location.name %>
                  </div>
                  <p class="icons">
                    <button class="btn talk__audience"><%= talk.audience.name %></button>
                    <%= for category <- talk.categories do %>
                        <%= if category.icon_url do %>
                          <img src="<%= category.icon_url %>" alt="<%= category.name %>">
                        <% else %>
                          <button class="btn talk__category">
                            <%= category.name %>
                          </button>
                        <% end %>
                    <% end %>
                  </p>
                  <%= if @conn.assigns[:current_user] do %>
                    <div class="schedule__action" data-talk-id="<%= talk.id %>">
                      <div class="add-to-schedule">
                        <i class="far fa-calendar-check"></i>
                        ADD TO AGENDA
                      </div>
                      <div class="remove-from-schedule" style="display: none">
                        <i class="far fa-trash-alt"></i>
                        REMOVE FROM AGENDA
                      </div>
                    </div>
                  <% end %>
                </div>
              <% end %> <!-- end for loop -->
            <% end %> <!-- end if statement -->
          </div>
        </div>

      <% end %>
    <% end %>
  </div>
</div>

<%= if @conn.assigns[:current_user] do %>
  <script>
    $(document).ready(function() {
      $('.add-to-schedule').click(function(event) {
        var parent = $(this).parent();
        var talkId = parent.data('talk-id');
        var data = { user_id: <%= @conn.assigns[:current_user].id %>, talk_id: talkId}

        $.post( '/api/users_talks', data)
            .done(function( data ) {
              console.log(data.data.id);
              console.log(parent.find('.remove-from-schedule'));
              parent.find('.add-to-schedule').hide();
              parent.find('.remove-from-schedule').data('user-talk-id', data.data.id).show();
            });
        })

      $('.remove-from-schedule').click(function(event) {
        var parent = $(this).parent();
        var id = $(this).data('user-talk-id');

        $.ajax({
            url: '/api/users_talks/' + id,
            type: 'DELETE',
            success: function(result) {
              parent.find('.add-to-schedule').show();
              parent.find('.remove-from-schedule').hide();
            }
        });
      });

    });
  </script>
<% end %>

```

Restart your server. Press `Cmd` + `c` to quit. Then run:

```
mix phx.server
```


## Exercise #1: Load speakers
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

4. Let's update the layout. Open `lib/fawkes_web/templates/speaker/index.html.eex`

      ```
      <div class="container speaker-index">
        <h4 class="subheader">Speakers</h4>
        <div class="row attendee">
          <%= for speaker <- @speakers do %>
              <div class="card col-xs-4">
                <div class="card-body ">
                  <img src="<%= speaker.image_url %>" class="rounded-circle">
                  <h4><%= link FawkesWeb.SharedView.full_name(speaker), to: speaker_path(@conn, :show, speaker) %></h4>
                  <p><%= link speaker.talk.title, to: "/talks/#{speaker.talk.id}" %></p>
                </div>
              </div>
          <% end %>
        </div>
      </div>
      ```

5. Go to [http://localhost:4000/speakers](http://localhost:4000/speakers), make sure the speaker loads


## Exercise #2: Show talk
(Use git checkout `5c.show_talk` to catch up with the class)

1. In `lib/fawkes/schedule/schedule.ex`, find the function `get_talk`. Preload slot, speakers, categories, audience, location
2. Open `lib/fawkes_web/templates/talk/show.html.eex`. Replace the content with this HTML:

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
          <img src="https://lh3.googleusercontent.com/-fEyw5rjNaN0/VIcChZfCYDI/AAAAAAAADfA/OW_u5voliss/w426-h351/good-job-job-lucky-star-konata-izumi-good-anime-otakus-1375040667.jpg" class="rounded-circle">
          <i class="fas fa-check-circle attendee__check"></i>
          <h4>Name name</h4>
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
