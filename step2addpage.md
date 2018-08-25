---
layout: page
title: Add a page
---


## Add a page
When a request is sent, Phoenix will look at the router file to figure out which controller handles the request. You can see all the routes by running `mix phx.routes` in the command line. To add a new endpoint, use this pattern: `[HTTP verb] [path], [controller], [function]`. Let's add an about page to our app. First we need to add a get path to our app.
  1. Open `lib/fawkes_web/router.ex`
  2. Under `pipe_through :browser`, on line 18, add the line below. This means when a GET request for `/about`, the `about` function in the `PageController` will handle the request.

    ```
      get "/about", PageController, :about
    ```

  3. Open `lib/fawkes_web/controllers/page_controller.ex`
  4. Add the function to handle the request after line 6:

    ```
      def about(conn, _params) do
        render conn, "about.html"
      end
    ```

  5. Create a new file `lib/fawkes_web/templates/page/about.html.eex`
  6. Add the following content to the file:

  ```
  <div class="section-white">
<div class="container">
  <div class="banner-wrapper">
    <img src="/images/elixirconf.png" class="banner">
  </div>
  <p class="text-center">4 - 7 September 2018  |  Bellevue, Washington</p>

  <hr />

  <div class="section">
    <p>
      ElixirConf® US is the premier conference for the Elixir community where Elixir developers from around the world gather.
    </p>

    <p>ElixirConf® is dedicated to advancing the Elixir programming language and the communities and companies surrounding it by bringing together the technically minded to establish relationships for work, collaboration, and entrepreneurship.</p>

    <p class="text-center">
      <a href="/schedules" class="btn">View the Schedule</a>
    </p>
  </div>
</div>
</div>
<div class="container section venue">
<h4 class="subheader">Hyatt Regency Bellevue, WA</h4>
<img src="/images/hyatt.png">
<h4 class="subheader">VENUE</h4>
<p>The Hyatt Regency Bellevue is the largest of any Bellevue hotel on the Eastside of Seattle. It has a state-of-the-art StayFit™ Gym and Lap swimming pool, plus connecting sky bridges to access more than 250 shops, 45 restaurants and lounges, and exciting entertainment venues.</p>

<p>ElixirConf attendees can book a room at the hotel for $177 per night until the room block is sold out!</p>

<h4 class="subheader">LOCATION</h4>

  <a href="https://www.google.com/maps/place/900+Bellevue+Way+NE,+Bellevue,+WA+98004/@47.6182836,-122.2032197,17z/data=!3m1!4b1!4m5!3m4!1s0x54906c8f16498957:0x985f05ca82fc9bc2!8m2!3d47.61828!4d-122.201031">900 Bellevue Way NE Bellevue, WA, USA, 98004-4272</a>

<iframe src="https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d2689.403690964853!2d-122.20321432016335!3d47.618283595266206!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x54906c8f16498957%3A0x83435bd9679eaf7!2sHyatt+Regency+Bellevue+on+Seattle's+Eastside!5e0!3m2!1sen!2sus!4v1491230376954" allowfullscreen="" width="600" height="450"></iframe>

<p>Located 20 minutes from Sea-Tac International Airport and nine miles east of Seattle, the Hyatt Regency is convenient to the corporate headquarters of Microsoft, Eddie Bauer, Symetra, Paccar, Expedia, Nintendo, and T-Mobile.</p>


<h4 class="subheader">TRANSPORTATION</h4>
<p>To get from the SEA Airport to the Hyatt Regency, the best options are lyft, Uber, taxi, or Shuttle Express.</p>

<p class="text-center">
  <a href="https://book.passkey.com/go/ELIXIRCONFERENCE2018" target="_blank" class="btn">Book Your Room</a>
</p>
</div>
  ```

  7. Create a new file named `assets/css/about.css`. Add the following content to style the about page:

  ```
  .banner-wrapper {
    text-align: center;
    margin-top: 20px;
    margin-bottom: 20px;
  }

  .banner {
    width: 390px;
  }
  ```

  8. Go to [http://localhost:4000/about](http://localhost:4000/about)

## Congratulations, you have a page!!!

<img src="http://wac.450f.edgecastcdn.net/80450F/thefw.com/files/2012/10/dancinggif.gif">
