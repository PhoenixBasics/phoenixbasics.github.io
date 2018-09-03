---
layout: page
title: API - Add talks to agenda
---

(Use `git checkout 8b.others_agenda` to catch up with the class)

1. Add a new endpoint `/agendas/:user_slug`

    ```
    get "/agendas/:user_slug", AgendaController, :show
    ```

2. Add a new function in the `lib/fawkes_web/controller/agenda_controller.ex`. We can reused some of the functions from earlier.

      ```
      alias Fawkes.Membership

      def show(conn, %{"user_slug" => user_slug}) do
        profile = Membership.fetch_user_profile(user_slug)
        talks = Agenda.list_talk_for_user(profile.user_id)
        render(conn, "index.html", talks: talks)
      end
      ```
