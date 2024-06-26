---
layout: post
title: "Using Grape to add an API to a brownfield Rails Project"
description: ""
excerpt: "A short journey of adding a grape based API to an existing Ruby on Rails project."
category: 
tags: [rails, grape, api]
---
{% include JB/setup %}

Starting point: We needed a small and simple API added to a several year
old rails 4.2 project. We already have some APIs in use in this project
which use basic rails controllers, ActiveModel, and sometimes jbuilder.
So quickest would be to just build the API as we did with all the
others. Maybe even use newish rails 5 ActionController::API and use
[json:api](http://jsonapi.org/) setting to have some kind of
standardization. Up to now we never needed real documentation (beside
the code) as we ourselves consumed the APIs. What seems a nice option
for API-docus is to use the test cases to generate the documentation as
does
[rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation)
or what can be done by writing cucumber files (see for example [cucumber
docs on
relishapp](http://www.relishapp.com/cucumber/cucumber/docs/cli/dry-run)
).

BUT: I heard a lightening talk at this years
[wroc_love.rb](http://www.wrocloverb.com/) by
[LeFnord](https://github.com/LeFnord) on the [grape api
gem](http://www.ruby-grape.org/) and his little baby
[grape-swagger](https://github.com/ruby-grape/grape-swagger) which
provides beautiful API documentation out of the box.

As our new API is to be consumed by a third Party we do need some
documentation this time. Also we have some time to spare and are looking
for a long term solution to our API-needs. We therefor decided to try
out grape which is a framework independend of rails and does seem to do
quite a lot of stuff in a reasonable manner :)

### Our first grape endpoint

First of to find out what we need to include to use grape inside a rails
project. There is a gem which wraps the varios best practices of
integrating grape into rails
([grape_ape_rails](http://mepatterson.github.io/grape_ape_rails/) ) but
it has not been maintained recently (last commit 2 years ago). Also it
propably is better to add what is needed step by step to get to know the
grape ecosystem. We therefor start of with adding

    gem 'grape'

which is version 0.18.0 in our case.

Next up we have a look at the README. Wow - pages over pages. A lot more
than the usual rails gems which consist of a five steps and finish
approach. A bit overwhelming at first, especially as one has to filter
out what's necessary for a rails project.

We create our first API endpoint (i call it that, found no other fitting
name) by placing a ruby file under app/api. We have to follow rails
conventions for naming classes/modules (i.e. filenames + folders) but we
do not need to create a v1 folder to version our API. Instead grape
allows us to define the version (and what strategy to use e.g. path or
header) inside our endpoint.

    class Nirvana < Grape::API
      version 'v1'
      format :json
      prefix :api

      resource :cities do
        desc 'Return ten cities matching the query'

        params do
          requires :q, type: String
        end
        get :all do
          City.search(params['q']).limit(10)
        end
      end

    end

For our first tryout we include a simple cities resource using a query
param.

Next up a test (yea next time we'll write that one first ;) ). we create
a test/api directory to bundle up everything api related. The test does
not differ much from usual model tests although this is more like an
rails controller test.

    require 'test_helper'

    class NirvanaTest < ActiveSupport::TestCase
      include Rack::Test::Methods

      def app
        Rails.application
      end

      test 'GET /api/statuses/public_timeline returns an empty array of statuses' do
        get '/api/v1/nirvana/cities'
        assert last_response.ok?
        assert_equal [], JSON.parse(last_response.body)
      end

    end

We need the app method so rack-test knows what to use. I wonder if there
will be a point at which it might be nice to have a stricter seperation
of testing as in rails (models, controllers, integration). Although an
API of course is diffent.

First test run of course fails. We have not yet mounted the API nor
loaded the app/api directory. Unfortunately test is still red. Now comes
the hard part. How do we debug Grape inside of rails? First guess is to
validate we actually are using the right route. Unfortunately our handy
'rake routes' does not help with this. We therefor inspect
'Nirvana.routes' on the console.

     > Nirvana.routes.map{|route| route.pattern.path}
     => ["/api/:version/cities/all(.json)"] 
     

Okay, so the route consists of api-prefix + version + resource + action.
I wonder if this is fixed or other parts may be inserted. It should be
easier to print out grape routes than it is today. I tried out the
[grape-rails-routes](https://github.com/pmq20/grape-rails-routes) gem
but it does not seem to work with recent grape versions. Better luck
with [grape-raketasks](https://github.com/reprah/grape-raketasks) which
works although the printout could be a bit more compact:

    $ rake grape_raketasks:routes
    ANCHOR:          true
    API:             Nirvana
    DESCRIPTION:     "Return ten cities matching the query"
    FORWARD_MATCH:   nil
    METHOD:          "GET"
    NAMESPACE:       "/cities"
    PARAMS:          {"q"=>{:required=>true, :type=>"String"}}
    PREFIX:          :api
    REQUIREMENTS:    {}
    SETTINGS:        {:description=>{:description=>"Return ten cities matching the query", :params=>{"q"=>{:required=>true, :type=>"String"}}}, :declared_params=>[:q]}
    SUFFIX:          "(.json)"
    VERSION:         "v1"

And thats for only one single endpoint - good luck with a real API.

### And now the documentation

Next up we want to generate some beautiful documentation for our API.
For this we wil be using the grape-swagger gem which generates the docs.
As we want to directly include them in our rails project we start off
with the
[grape-swagger-rails](https://github.com/ruby-grape/grape-swagger-rails)
gem which allows us to mount the swagger UI as an engine. So off we go,
follow the README, add gem, create initializer and open the swagger
UI...

![Swagger UI is not working yet]({{ site.url }}/assets/images/grape/grape-swagger-rails-ui-cant-read.png "Swagger UI is not working yet")

Hmm... only kinda works :) We had to specify an url and app_url in an
initializer. The example url was just a path to a json file. I guess
it's supposed to be the 'Swagger API schema'. Question is how to
generate this. Seems as if swagger itself is not included in the
grape-swagger-rails gem which in my opinion does not make sense or at
least is not what one would expect from a rails gem.

So let's install
[grape-swagger](https://github.com/ruby-grape/grape-swagger) and set it
up. We follow the README and create a root node:

    require 'grape-swagger'

    class Root < Grape::API
      mount Nirvana
      add_swagger_documentation
    end

\
But the suggested path 'localhost:3000/swagger_doc' does not work for
us. We just end up with an 404 from our rails app. Our rake grape routes
gem also gives no hint as to which path to use. So after trying out a
whole lot of different routes (for instance
'localhost:3000/api/v1/swagger_doc') we still do not know how to
actually generate the swagger schema. Then lightening struck - the
example given by grape-swagger mounts a couple of other endpoints. I
thought this was only due to whats supposed to be included in the docs
but then i realized the Root node is the one which should be mounted
inside routes.rb. Only then the swagger_doc is accessible. For now the
easiest is to simply include the 'add_swagger_documentation' into our
initial cities endpoint. Five seconds later after changing the schema
url to 'api/v1/swagger_doc':

![Yeah]({{ site.url }}/assets/images/grape/grape-swagger-ui-working.png "Yeah")

Nice! A usable and intuitive documentation for our API. Some polishing
still has to be done (for instance where will we set 'API title'?) and
some details are still puzzeling (i.e. why is there a 0.0.1 version
shown in the footer when we specified v1 as our API version?). But all
in all it's great :)

### Conclusion

So all in all we have a nice API with great documentation generated for
us. On the downside it is a pitty many of our trusted rails tools do not
work and/or have to be reinvented. We will see how much overhead this
adds to further developing the API and adding more advanced features
such as authentication, caching or using JSON:API as format.
