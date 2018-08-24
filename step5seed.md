---
layout: page
title: Populate the database
---


Now that we have our modules, let's connect them all together?

Make a direction in `schedule` called `seed`

```
mkdir lib/fawkes/schedule/seed
```

Run this command to download the file to seed

```
curl -o lib/fawkes/schedule/seed/audience.ex https://raw.githubusercontent.com/mcelaney/fawkes/master/lib/fawkes/schedule/seed/audience.ex && curl -o lib/fawkes/schedule/seed/category.ex https://raw.githubusercontent.com/mcelaney/fawkes/master/lib/fawkes/schedule/seed/category.ex && curl -o lib/fawkes/schedule/seed/event.ex https://raw.githubusercontent.com/mcelaney/fawkes/master/lib/fawkes/schedule/seed/event.ex && curl -o lib/fawkes/schedule/seed/location.ex https://raw.githubusercontent.com/mcelaney/fawkes/master/lib/fawkes/schedule/seed/location.ex && curl -o lib/fawkes/schedule/seed/slot.ex https://raw.githubusercontent.com/mcelaney/fawkes/master/lib/fawkes/schedule/seed/slot.ex && curl -o lib/fawkes/schedule/seed/speaker.ex https://raw.githubusercontent.com/mcelaney/fawkes/master/lib/fawkes/schedule/seed/speaker.ex && curl -o lib/fawkes/schedule/seed/talk.ex https://raw.githubusercontent.com/mcelaney/fawkes/master/lib/fawkes/schedule/seed/talk.ex && curl -o lib/fawkes/schedule/seed/seed.ex https://raw.githubusercontent.com/mcelaney/fawkes/master/lib/fawkes/schedule/seed/seed.ex
```

Add this line to `priv/repo/seeds.exs` run the seed file:

```
Fawkes.Schedule.seed()
```

Let's seed our database:

```
mix run priv/repo/seeds.exs
```
