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

## Fetching data with JavaScript

While it might be tempting to deploy this same strategy for our "Country" and
"State" `<select>` element pairing, rendering all possible combinations of
Country and State would be far too expensive:

```ruby
irb(main):001:0> country_codes = CS.countries.keys
=>
[:AD,
...
irb(main):002:0> country_codes.flat_map { |code| CS.states(code).keys }.count
=> 3391
```

Instead, we can render a single pairing of Country and State values, then fetch
a new pair when the `<select>` element's selected options change.

A `<button formmethod="get">` element powers the original (JavaScript-free)
version of State options fetching. End-users manually click the `<button>` to
fetch new options. In cases where the visiting browsing environment has
JavaScript disabled, we'll continue to support that behavior.

To do so, we'll nest the `<button>` within a [`<noscript>` element][noscript] so
that it's present with JavaScript enabled, but absent otherwise:

[noscript]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/noscript

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
     <%= form.label :country %>
     <%= form.select :country, @address.countries.invert %>

+    <noscript>
       <button formmethod="get" formaction="<%= new_address_path %>">Select country</button>
+    </noscript>
```

In it's place, we'll introduce a visually-hidden `<button>` element
counterpart that our application's JavaScript code will programmatically click
on the end-user's behalf:

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
     <%= form.label :country %>
     <%= form.select :country, @address.countries.invert %>

+    <button formmethod="get" formaction="<%= new_address_path %>" hidden></button>
     <noscript>
       <button formmethod="get" formaction="<%= new_address_path %>">Select country</button>
     </noscript>
```

We'll introduce a Stimulus controller to monitor changes to the `<select>`
element's selected options. Whenever a [change][] event fires on the `<select>`
element, we'll programmatically click the `<input type="submit">`.

To scope the event monitoring to this cluster of fields, we'll nest the
`<label>`, `<select>`, and `<input type="submit">` elements within a
`<fieldset>` element that declares the `[data-controller]` attribute to contain
the `element` identitier. We'll declare the `<fieldset>` element to have
[`display: contents`][contents] so that its descendants can continue to
participate in the `<fieldset>` element's flexbox layout:

[contents]: https://developer.mozilla.org/en-US/docs/Web/CSS/display#box
[change]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/change_event

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
+    <fieldset class="contents" data-controller="element">
       <%= form.label :country %>
       <%= form.select :country, @address.countries.invert %>

       <button formmethod="get" formaction="<%= new_address_path %>" hidden></button>
       <noscript>
         <button formmethod="get" formaction="<%= new_address_path %>">Select country</button>
       </noscript>
+    </fieldset>
```

Next, we'll mark the `<input type="submit">` element with the
`[data-element-target="click"]` attribute so that the `element` controller can
access the element directly. We'll also route `change` events fired by the
`<select>` element to the `element#click` action:

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
     <fieldset class="contents" data-controller="element">
       <%= form.label :country %>
-      <%= form.select :country, @address.countries.invert %>
+      <%= form.select :country, @address.countries.invert, {},
+                      data: { action: "change->element#click" } %>

-      <button formmethod="get" formaction="<%= new_address_path %>" hidden>
+      <button formmethod="get" formaction="<%= new_address_path %>" hidden
+              data-element-target="click"></button>
       <noscript>
         <button formmethod="get" formaction="<%= new_address_path %>">Select country</button>
       </noscript>
     </fieldset>
```

The `element` controller's implementation is minimal and extremely specific.
Clicking its "click" targets is its one and only behavior:

```javascript
// app/javascript/controllers/element_controller.js

import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "click" ]

  click() {
    this.clickTargets.forEach(target => target.click())
  }
}
```

Similar to our `<input type="radio">` monitoring implmentation, it's important
to render the `<select>` element with [autocomplete="off"][] so that
browser-initiated optimizations don't introduce inconsistencies between the
initial client-side selection and the server-rendered selection:

[autocomplete="off"]: https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/autocomplete#values

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
       <%= form.label :country %>
-      <%= form.select :country, @address.countries.invert, {},
+      <%= form.select :country, @address.countries.invert, {}, autocomplete: "off",
                       data: { action: "change->element#click" } %>
```

https://user-images.githubusercontent.com/2575027/150692642-39273c3c-370d-468a-aa5a-49b9cbac1120.mov

## Fetching data with Turbo Frames

While we've enhanced our JavaScript-free solution for maintaining
synchronization between the "Country" and "State" `<select>` elements, there are
still some quirks to improve.

For example, because the `<form>` submission triggers a full-page navigation,
our application will discard any client-side state like, which element has focus
or how far down the page the end-user has scrolled. Ideally, a change to the
"Country" `<select>` element's current option would fetch and replace an HTML
fragment that _only_ contained the "State" `<select>` element, and left the rest
of the page undisturbed.

Lucky for us, [Turbo Frames][] are well suited for that purpose! Through the
[`<turbo-frame>`][] custom element (and its [src][turbo-frame-src] attribute),
our application can manage fragments of our document asynchronously.

We can scope our "Country" `<select>`-driven requests the portion of our from that contains the "State" `<select>` and `<label>` elements.

First, we'll wrap the "State" portion of the form in a
[`<turbo-frame>`][turbo-frame] element, and assign it an `[id]` attribute that
is unique across the document:

[Turbo Frames]: https://turbo.hotwired.dev/handbook/frames
[turbo-frame]: https://turbo.hotwired.dev/reference/frames#basic-frame
[turbo-frame-src]: https://turbo.hotwired.dev/reference/frames#html-attributes
[FrameElement]: https://turbo.hotwired.dev/reference/frames#properties
[data-turbo-frame]: https://turbo.hotwired.dev/handbook/frames#targeting-navigation-into-or-out-of-a-frame

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
+    <turbo-frame id="<%= form.field_id(:state, :turbo_frame) %>" class="contents">
       <% if @address.states.any? %>
         <%= form.label :state %>
         <%= form.select :state, @address.states.invert %>
       <% end %>
+    </turbo-frame>
```

There are several ways to navigate the frame. For example, if it better suits
your use-case, you can retrieve the [FrameElement][] instance, and interact with
the `[src]` property directly.

Since we're already rendering an `<input type="submit">` element that
declaratively encodes all the pertinent, concrete details of the submission (for
example, the `[formmethod]` and `[formaction]` attributes), we'll rely on the
fact that we're programmatically clicking the `<input type="submit">` element.

Not only does the `<input type="submit">` encode _where_ and _how_ to make our
submission, the browser's built-in form field encoding mechanisms to control
_what_ to include in the submission.

To target and drive the `<turbo-frame>` element, we'll render the `<input
type="submit">` element with a [data-turbo-frame][] attribute that references
the `<turbo-frame>` element's `[id]` attribute:

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
       <button formmethod="get" formaction="<%= new_address_path %>" hidden
-              data-element-target="click"></button>
+              data-element-target="click" data-turbo-frame="<%= form.field_id(:state, :turbo_frame) %>"></button>
       <noscript>
         <button formmethod="get" formaction="<%= new_address_path %>">Select country</button>
       </noscript>
```

With those enhancements in place, changes to the "Country" `<select>` element
refreshes the "State" `<select>` element's collection of options while
retaining client-side state like which element is focused (in this case, the
"Country" `<select>` _retains_ focus throughout):

https://user-images.githubusercontent.com/2575027/150692695-a9cc60c7-0fbf-4e52-8dd9-a9b8950f8210.mov

Now that we're navigating a fragment of the page, there's an opportunity to
reduce the amount of data we're encoding into the `GET` request. In our
example's case, we're unlikely to exceed the URL's 2,000 character limit.
However, that might not be true for every use case. As an alternative to
submitting _all_ of the `<form>` element's fields' data to the `<turbo-frame>`,
we'll introduce a `search-params` controller to only encode the _changed_
field's data into an `<a>` element's [href][] attribute.

[href]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#attr-href

First, we'll replace the visually hidden `<input type="submit">` with a visually
hidden `<a>` element, and mark that element with the
`[data-search-params-target="anchor"]` attribute:

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
-      <button formmethod="get" formaction="<%= new_address_path %>" hidden
-              data-element-target="click" data-turbo-frame="<%= form.field_id(:state, :turbo_frame) %>"></button>
+      <a href="<%= new_address_path %>" data-search-params-target="anchor" hidden
+              data-element-target="click" data-turbo-frame="<%= form.field_id(:state, :turbo_frame) %>"></a>
```

In our template, we can add `search-params` to the list of tokens declared on
our `<div data-controller="element">` element. In this case, the order that the
tokens are declared is not significant:

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
-    <fieldset class="contents" data-controller="element">
+    <fieldset class="contents" data-controller="search-params element">
       <%= form.label :country %>
       <%= form.select :country, @address.countries.invert, {}, autocomplete: "off",
                       data: { action: "change->element#click" } %>
```

Next, we'll add a `change->search-params#encode` action routing descriptor to
the `<select>` element's `[data-action]` attribute:

```diff
--- a/app/views/addresses/new.html.erb
+++ b/app/views/addresses/new.html.erb
     <fieldset class="contents" data-controller="search-params element">
       <%= form.label :country %>
       <%= form.select :country, @address.countries.invert, {}, autocomplete: "off",
-                      data: { action: "change->element#click" } %>
+                      data: { action: "change->search-params#encode change->element#click" } %>

       <a href="<%= new_address_path %>" data-search-params-target="anchor" hidden
               data-element-target="click" data-turbo-frame="<%= form.field_id(:state, :turbo_frame) %>"></a>
```

The order that the tokens are declared **is significant**. According to the
Stimulus documentation for [declaring multiple actions][multiple-actions]:

> When an element has more than one action for the same event, Stimulus invokes
> the actions from left to right in the order that their descriptors appear.

In our case, we need `search-params#encode` to precede `element#click`, so that
the value is encoded into the `<a href="...">` attribute before we navigate the
related `<turbo-frame>` element.

[multiple-actions]: https://stimulus.hotwired.dev/reference/actions#multiple-actions

The `search-params` controller's `encode` action transforms the target element's
`name` and `value` properties into a [URLSearchParams][] instance's key-value
pair, then assigns it to the [HTMLAnchorElement.search][] property across its
collection of `anchorTargets`:

[URLSearchParams]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams
[HTMLAnchorElement.search]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLAnchorElement/search

```javascript
// app/javascript/controllers/search_params_controller.js

import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "anchor" ]

  encode({ target: { name, value } }) {
    for (const anchor of this.anchorTargets) {
      anchor.search = new URLSearchParams({ [name]: value })
    }
  }
}
```

With those changes in place, the `<a>` element's `[href]` attribute only encodes
the pertinent values (e.g. `/buildings/new?building%5Bcountry%5D=US`).
