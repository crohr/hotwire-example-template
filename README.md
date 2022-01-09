# Hotwire: Dynamic form fields with Turbo

[![Deploy to Heroku](https://www.herokucdn.com/deploy/button.png)][heroku-deploy-app]

[heroku-deploy-app]: https://heroku.com/deploy?template=https://github.com/thoughtbot/hotwire-example-template/tree/hotwire-example-turbo-dynamic-forms

One common problem present in interactive web applications is a need to fetch
content or data from the server.

Imagine a page to collect a shipping information. The form could present text
fields to collect the address' street number, city, and postal code, and lists
of options to collect the state and country.

To start, we'll establish a baseline that renders HTML retrieved over HTTP
without any JavaScript. The form will present the states within the United
States. Next, we'll collect the address' country, which will update the list of
states to choose from.

Once we've established a foundation, we'll progressively enhancement the form's
interactivity to enable the "passcode" field when "passcode protected" is
selected, and disable and hide it otherwise. This version will use full-page
navigations and round-trips to the server to fetch updated HTML, and will work
even when JavaScript is disabled.

Finally, we'll incrementally improve the experience through JavaScript that
controls the "passcode" field without any client-server communication.

The code samples shared in this article omit the majority of the application’s
setup. The application's code was generated by executing `rails new`. The rest
of the [source code][] from this article (including a [suite of tests][]) can be
found on GitHub, and is best read [commit-by-commit][].

[source code]: https://github.com/thoughtbot/hotwire-example-template/tree/hotwire-example-turbo-dynamic-forms
[suite of tests]: https://github.com/thoughtbot/hotwire-example-template/tree/hotwire-example-turbo-dynamic-forms/test
[commit-by-commit]: https://github.com/thoughtbot/hotwire-example-template/compare/hotwire-example-turbo-dynamic-forms

## Our starting point

We'll render a form that collects information about `Address` records. We're
interested in the address and whether it's "owned", "leased", or "other". When
it's "leased", we'll require that the submission includes a management phone
number. When it's "other", we'll require a description. Otherwise, both fields
are optional.

A `Address` record's `country` column will default to the United States (that
is, a `country` attribute with a value of `"US"`). We're relying on the
[city-state][] gem to provide our form with a collection of "Country" and
"State" options.

In addition to validations, the `Address` model class defines an
[enumeration][] and some convenience methods to access Countries and States
provided by the `city-state` gem (invoked about through the `CS` class):

[city-state]: https://github.com/loureirorg/city-state/
[enumeration]: https://edgeapi.rubyonrails.org/classes/ActiveRecord/Enum.html

```ruby
class Address < ApplicationRecord
  with_options presence: true do
    validates :line_1
    validates :city
    validates :postal_code
  end

  validates :state, inclusion: { in: -> record { record.states.keys }, allow_blank: true },
                    presence: { if: -> record { record.states.present? } }

  def countries
    CS.countries.with_indifferent_access
  end

  def country_name
    countries[country]
  end

  def states
    CS.states(country).with_indifferent_access
  end

  def state_name
    states[state]
  end
end
```

The `addresses/new` template collects values and submits the `<form>` as a
`POST` request to the `AddresssController#create` action:

```erb
<%# app/views/addresses/new.html.erb %>

<section class="w-full max-w-lg">
  <h1>New address</h1>

  <%= form_with model: @address, class: "flex flex-col gap-2" do |form| %>
    <%= form.label :line_1 %>
    <%= form.text_field :line_1 %>

    <%= form.label :line_2 %>
    <%= form.text_field :line_2 %>

    <%= form.label :city %>
    <%= form.text_field :city %>

    <%= form.label :state %>
    <%= form.select :state, @address.states.invert %>

    <%= form.label :postal_code %>
    <%= form.text_field :postal_code %>

    <%= form.button %>
  <% end %>
</section>
```

![A form collecting information about an Address](https://user-images.githubusercontent.com/2575027/150692333-295fb94f-8f02-4f48-a766-f8c485e8ecc7.png)

`Address` records are managed by a conventional `AddressesController` class:

```ruby
# app/controllers/addresses_controller.rb

class AddressesController < ApplicationController
  def new
    @address = Address.new
  end

  def create
    @address = Address.new address_params

    if @address.save
      redirect_to address_url(@address)
    else
      render :new, status: :unprocessable_entity
    end
  end

  def show
    @address = Address.find params[:id]
  end

  private

  def address_params
    params.require(:address).permit(
      :line_1,
      :line_2,
      :city,
      :state,
      :postal_code,
    )
  end
end
```

When the submission is valid, the record is created, the data is written to the
database, and the controller serves an [HTTP redirect response][redirect] to the
`addresses#show` route. When the submission's data is invalid the controller
re-renders the `bulidings#new` template, responds with a [422 Unprocessable
Entity][422].

[422]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/422
[redirect]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Redirections

## Interactivity and dynamic options

Our starting point serves as a solid, reliable, and robust foundation. The
"moving parts" are kept to a minimum. The form collects information with or
without the presence of a JavaScript-capable browsing environment.

With that being said, there is still an opportunity to improve the end-user
experience. We'll start with a JavaScript-free baseline, then we'll
progressively the form, adding dynamism and improving its interactivity along
the way.

To start, let's support `Address` record in Countries outside the United
States. We'll add a `<select>` to our provide end-users with a collection of
Country options:

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
   <%= form_with model: @address, class: "flex flex-col gap-2" do |form| %>
     <%= render partial: "errors", object: @address.errors %>

+    <%= form.label :country %>
+    <%= form.select :country, @address.countries.invert %>
+
     <%= form.label :line_1 %>
```

Along with the new field, we'll add a matching key name to the
`AddresssController#address_params` implementation to read the new value from
a submission's parameters:

```diff
--- a/app/controllers/addresses_controller.rb
+++ b/app/controllers/addresses_controller.rb
@@ -20,7 +20,8 @@ class AddressesController < ApplicationController
   private

   def address_params
     params.require(:address).permit(
+      :country,
       :line_1,
       :line_2,
       :city,
```

While the new `<select>` provides an opportunity to pick a different Country,
that choice won't be reflected in the `<form>` element's collection of States.

What tools do we have at our disposal to synchronize the "States" `<select>`
with what's chosen in the "Countries" `<select>`? How might we fetch new
`<select>` and `<option>` elements from the server without without using
[XMLHttpRequest][], [fetch][], or any JavaScript at all?

[XMLHttpRequest]: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest
[fetch]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API

### Fetching data without JavaScript

Browsers provide a built-in mechanism to submit HTTP requests without JavaScript
code: `<form>` elements. By clicking `<button>` and `<input type="submit">`
elements, end-users submit `<form>` elements and issue HTTP requests. What's
more, those `<button>` elements are capable of overriding _where_ and _how_ that
`<form>` element transmits its submission by through their [formmethod][] and
[formaction][] attributes.

We'll change our `<form>` to present a "Select country" `<button>` element to
refresh the page's "State" options:

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
     <%= form.label :country %>
     <%= form.select :country, @address.countries.invert %>
+
+    <button formmethod="get" formaction="<%= new_address_path %>">Select country</button>
```

The `<button>` element's `[formmethod="get"]` attribute directs the `<form>` to
submit as an [HTTP GET][] request and the `[formaction="/addresses/new"]`
attribute directs the `<form>` to submit to the `/addresses/new` path. This
verb-path pairing might seem familiar: it's the same request our browser will
make when we visit the current page.

Submitting `<form>` as a `GET` request will encode all of the fields' values
into [URL parameters][]. We can read those values in our `addresses#new` action
whenever they're provided, and use them when rendering the `<form>` element and
its fields:

```diff
--- a/app/controllers/addresses_controller.rb
+++ b/app/controllers/addresses_controller.rb
 class AddresssController < ApplicationController
   def new
-    @address = Address.new
+    @address = Address.new address_params
   end

   def create
@@ -20,7 +20,7 @@ class AddresssController < ApplicationController
   private

   def address_params
-    params.require(:address).permit(
+    params.fetch(:address, {}).permit(
       :country,
       :line_1,
       :line_2,
       :city,
       :state,
       :postal_code,
     )
   end
 end
```

https://user-images.githubusercontent.com/2575027/150692412-47d523dd-b4c6-4e1c-9324-909a8bff4f4c.mov

It's worth noting that there are countries that don't have "State" options (like
Vatican City), so we'll also want to account for that case in our
`addresses/new` template:

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
+    <% if @address.states.any? %>
       <%= form.label :state %>
       <%= form.select :state, @address.states.invert %>
+    <% end %>
```

https://user-images.githubusercontent.com/2575027/150692444-f69bc3c1-5366-4a66-a398-f273e90b2a8d.mov

Submitting the form's values as query parameters comes with two caveats:

1.  Any selected `<input type="file">` values will be discarded

2.  according to the [HTTP specification][], there are no limits on the length of
    a URI:

    > The HTTP protocol does not place any a priori limit on the length of
    > a URI. Servers MUST be able to handle the URI of any resource they
    > serve, and SHOULD be able to handle URIs of unbounded length if they
    > provide GET-based forms that could generate such URIs.
    >
    > - 3.2.1 General Syntax

    Unfortunately, in practice, [conventional wisdom][] suggests that URLs over
    2,000 characters are risky.

In the case of our simple example `<form>`, neither points pose any risk.

[HTTP specification]: https://tools.ietf.org/html/rfc2616#section-3.2.1
[conventional wisdom]: https://stackoverflow.com/a/417184
[URL parameters]: https://developer.mozilla.org/en-US/docs/Learn/Common_questions/What_is_a_URL#parameters
[formmethod]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/button#attr-formmethod
[formaction]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/button#attr-formaction
[HTTP GET]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET
