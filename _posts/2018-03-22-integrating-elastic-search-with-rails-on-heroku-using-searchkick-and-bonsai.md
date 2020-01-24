---
layout: post
title:  "Integrating Elastic Search with Rails on Heroku using Searchkick and Bonsai"
date:   2018-03-22 08:12:43 -0700
categories: ruby, rails, elasticsearch, heroku, searchkick, bonsai
---

If you’re reading this, I’ll assume you’ve already decided it’s much smarter not to build your own search engine from scratch in a slow language like Ruby simply to add some basic search functionality to one of your Rails models. If that’s true, wise decision! I won’t bore you with a wordy introduction to the context in which I decided to not roll out my own search engine from scratch. All you need to know is…

I have a Listing model that has a title, description, many tags, and belongs to an Artist.
I want users to be able to search for listings by keyword phrase.
The results should include close matches for both the title and description columns. They should also include close matches based on a listing’s tags, as well as it’s owners name (from the Artist model)
The results should be ordered based on a score of how closely they match the query across all 3 of those attributes. They should also be LIMITed and OFFSET for pagination.
Alright, let’s jump in.

## Adding Elastic Search to My App with Searchkick

The Searchkick gem makes it ridiculously easy to add Elastic Search to one of your models and keep the index of your records up to date. It also provides a nice wrapper API that’s SQL-like. Given that I know SQL, but don’t know the Elastic Search API, this is plenty enough reason for me to use it.

Before we use it though, we’ll have to install elasticsearch and have it listen on an open port. As always, Homebrew makes this super easy.

```bash
$ brew update
$ brew install elasticsearch
$ brew services start elasticsearch
```

Now let’s add Searchkick. Add this like to your Gemfile and run bundle.

```ruby
gem 'searchkick'
```

Now, we’re ready to add it to our model, in my case being Listing.

```ruby
class Listing << ApplicationRecord
  searchkick # that's all
  ...more code
end
```

## Constructing Our Query

Remember, we want the search to match across a listing’s title and description, but also it’s tags and owner’s name. This means that we want to keep the index up to date not only when a new listing is created, but also when a new tag is added to a listing, or when it’s owner changes their name (otherwise we’d have to do a full reindex before every search). Thankfully, Searchkick makes this easy too.

Just add a search_import scope to your model that includes the method(s) that reference the has_many and belongs_to relationships.

```ruby
scope :search_import, -> { includes(:tags, :artist) }
```

Now, any update to one of these peripheral attributes will update the elastic search index. We’ll also want to define a method called search_data that will describe how the data should be indexed.

```ruby
def search_data
  { artist_name: artist.name,
    title: title,
    description: description,
    tagged: "#{tags.map(&:name).join(' ')}" }
end
```

With that done, our only remaining demands are: set the fields the query should match on, order the results based on their match score, limit and offset the results for pagination.

Here’s what the query looks like:

```ruby
elastic_query = {
  fields: [:title, :description, :tagged, :artist_name],
    order: { _score: :desc },
    page: params[:page],
    per_page: 15
}
```

Pass this query to the .search class method that Searchkick added to our model, along with the search phrase itself and we’re in business.

```ruby
self.search(search_phrase, elastic_query)
```

## Cutting Out Unnecessary Calls

More often than not, you’ll want your users to be able to filter records in various ways outside of keyword search.

For example: in my app, in addition to listings being searched by keyword, they can also be filtered by category, city, distance from location etc.

I’d like all of this to be handled by one public class method so all I have to do in my controller is pass the full params hash and get the appropriate records back. I’ll show you how I handled this in my model, but the important part here is that you apply the other filters to your search while also avoiding making unnecessary calls to your Elasticsearch endpoint.

Note: You might ask, why not just have elastic search handle these other filters as well? The way I see it, you’re gonna use ActiveRecord to return the object representations of the records at the end regardless. Might as well avoid the elastic search call if you can. This also allows you to be selective with the columns that you have elastic search store in it’s index (something I haven’t done in this example). Also, in my case, I’m using the geocoder gem for searching by distance from current location. While elastic search does provide a way to search based on distance from coordinates, I’ve already implemented this in my app and would rather not rewrite that.

Here’s my code.

```ruby
class << self
  def query(params)
    permitted_params = params.slice(:page, :category, :search, :city, :range)
    listings = self.active_record_search(permitted_params) # filter by other parameters first

    # return right there if search is blank
    return listings.page(permitted_params[:page]) if permitted_params[:search].blank?

    # otherwise pass already filtered set to elastic search for further filtering
    listing_ids = listings.pluck(:id)
    self.elastic_search(permitted_params, listing_ids)
  end

  def active_record_search(params)
    # ...some code
  end

  def elastic_search(params, listing_ids)
    elastic_query = {
      fields: [:title, :description, :tagged, :artist_name],
      # include a where clause so I'm only searching records returned by the other filters
      where: { id: listings_ids },
      order: { _score: :desc },
      page: params[:page],
      per_page: 15
    }

    self.search(params[:search], elastic_query)
  end
end
```

As you can see (denoted by my comments), I first filter the records by the other parameters. I then check to see if there’s a search parameter. If not, I just return those records. If there is a search parameter, I pass those already filtered records along to the `elastic_search` method.

The only other thing to notice here is that you have to add the where clause to your elastic search query where { id: listings.map(&:id) } so you aren’t searching the entire listings table and disregarding the other filters.

Deploying to Heroku
Now we get to the part which is always scariest when implementing a new feature that relies on running another process outside of your web application: deploying to production.

Thankfully, Heroku usually has a simple solution in the form of an addon and this case is no exception. There’s a Bonsai addon that handles the elasticsearch service so you don’t have to do a damn thing.

From your terminal run…

```bash
$ heroku addons:create bonsai
```

That’ll automatically set an environment variable on Heroku called BONSAI_URL which is the endpoint you’ll make your elastic search calls to.

The only thing left to do is add an initializer that uses this endpoint in production.

Create a new file in your config/initializers directory called elasticsearch.rb and add this code to it.


Finally, build the index.

```bash
$ heroku run rails searchkick:reindex CLASS=Listing
```

Done!

You’re up and running with search that’s far better and faster than anything you could have written yourself and won’t require major changes as you add to the complexity of your queries in the future.

You’re little app is now practically Google.
