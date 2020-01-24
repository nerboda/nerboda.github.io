---
layout: post
title:  "Writing Sinatra Extensions"
date:   2017-01-26 09:04:43 -0700
categories: ruby, sinatra
---

I was recently messing around with Sinatra (the Ruby Web App DSL) and wanted to gain a little deeper insight into how it's designed. In order to do this, I decided to build a Sinatra Extension (basically a modularized library that adds functionality to the Sinatra::Base class).

Enter EasyBreadcrumbs, a Ruby Gem that provides a single `easy_breadcrumbs` helper method to auto generate bootstrap breadcrumb html for every page of your Sinatra App, and can handle a wide range of different types of routes.

* [Github Repo](https://github.com/nerboda/easy_breadcrumbs)
* [Ruby Gem](https://rubygems.org/gems/easy_breadcrumbs)

## Building the Extension

First I read Sinatra's fantastic little guide on writing extensions, which I strongly suggest you read - [http://www.sinatrarb.com/extensions.html](http://www.sinatrarb.com/extensions.html).

If you don't feel like reading it, here's a summary...

### Sinatra allows for 2 different styles of applications.

**1) The "Classic" style, which is what you're using when you simply `require 'sinatra'` and start writing routes, helper methods, setting configuration etc, often all within a single file.**

When you `require 'sinatra'`, you're requiring a file that then requires `sinatra/main.rb`, which defines a subclass of the `Sinatra::Base` class called **"Application"**. This `Sinatra::Application` class is the context within which methods like `get`, `post`, `before` and  `configure` are evaluated. They're just class methods defined in the `Sinatra::Base` class which are exported to the top level.

Within the blocks passed to those methods, code is evaluated within the scope of a new instance of the `Sinatra::Base` class. You can get a clearer picture of this by throwing a `require 'pry'; binding.pry` inside a `get` route block, navigating to that route in your browser and then checking out `self` within the pry session. If you look at `self.object_id`, you'll see that it's a new object with each request.

**2) The Modular style, where you create a new class that inherits from Sinatra::Base.**

Rather than loading up the `sinatra/main.rb` file, which automatically creates the application for you, with the modular style you `require 'sinatra/base'` and explicitly create your own subclass of it. This is usually the choice for applications that are too large for the single file approach or that you want to package as a library or component within a larger rack based system.

Just remember, it's still the same underlying class for both styles (Sinatra::Base) so apart from some configuration options that are set in `sinatra/main.rb`, and the fact that modular style apps require you to explicitly register extensions (rather than them automatically being registered when the file is loaded) everything else is the same.

### 3 Easy Steps to Extending Sinatra

So here's what it looked like for me to add a new helper method to Sinatra and make it readily available within view or layout files to anyone who installs the gem.

{% highlight ruby %}
require 'sinatra/base'
require 'easy_breadcrumbs/version'
require 'easy_breadcrumbs/breadcrumb'

module Sinatra
  include ::EasyBreadcrumbs

  module EasyBreadcrumbs
    Breadcrumb = ::EasyBreadcrumbs::Breadcrumb

    def easy_breadcrumbs
      # Path from current Rack::Request object
      request_path = request.path

      # All GET request routes
      route_matchers = self.class.routes['GET'].map { |route| route[0] }

      # The rest is handled by Breadcrumb class
      breadcrumb = Breadcrumb.new(request_path, route_matchers)
      breadcrumb.to_html
    end
  end

  # Register helper with Sinatra::Application
  helpers EasyBreadcrumbs
end
{% endhighlight %}

**Step 1: Open up the Sinatra Module**

I `require 'sinatra/base'` then open up the module on line 5.

**Step 2: Define your Extension module**

This is done on line 8. The module includes a single `easy_breadcrumbs` method which simply passes the request path (available through the `Rack::Request` object) and the routes defined in the Sinatra app as arguments to the initialize method in Breadcrumb.

**Step 3: Register the methods contained in the module as helper methods with the Sinatra app using the `Sinatra.helpers` class method.**

That's all there is to it, well the extending Sinatra part anyways. I'll be writing more about packaging extensions as Ruby gems and how to write integration tests for those gems. Stay tuned!