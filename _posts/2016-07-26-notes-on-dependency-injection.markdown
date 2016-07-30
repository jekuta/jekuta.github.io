---
layout: post
title:  Notes on Dependency Injection
date:   2016-07-25 6:00:00 +0100
categories: ruby
---

*This post contains notes for the lecture "Improve your code with dependency injection" by Stephen Best*

| Lecture  | [link](https://skillsmatter.com/skillscasts/4437-improve-your-code-with-dependency-injection) |
| Slides   | [link](https://speakerdeck.com/bestie/improve-your-ruby-code-with-dependency-injection)       |

#### Definition

What is dependency injection? - "the practice of injecting the dependencies to class instead of
hardcoding them in the class"

#### Benefits:

1. Enables truly isolated unit tests
2. Enables re-usability of individual objects
3. Looses coupling

#### Example #1

Original class
{% highlight ruby %}
class Fuit < ActiveRecord::Base
  def self.in_season
    date = Date.today # Hardcoded dependency

    all
      .where(["season_start <= ?", date])
      .where(["season_end   <= ?", date])
  end
end
{% endhighlight %}

Original spec
{% highlight ruby %}
RSpec.describe Fruit do
  describe '.in_season' do
    before { seed_fruits }

    context 'in summer time' do
      # Monkey patching the hardcoded dependency
      before { allow(Date).to receive(:today).and_return('2013-07-07') }

      subject { Fruit.in_season }

      it { is_expected.to include(summer_fruits) }
      it { is_expected.not_to include(winter_fruits) }
    end
  end
end
{% endhighlight %}

A small but crucial change
{% highlight ruby %}
class Fuit < ActiveRecord::Base
  def self.in_season(date: Date.today) # Inject the dependency
    all
      .where(["season_start <= ?", date])
      .where(["season_end   <= ?", date])
  end
end
{% endhighlight %}

The test becomes less complex
{% highlight ruby %}
RSpec.describe Fruit do
  describe '.in_season' do
    before { seed_fruits }

    context 'in summer time' do
      let(:date) { '2013-07-07' }

      subject { Fruit.in_season(date) }

      it { is_expected.to include(summer_fruits) }
      it { is_expected.not_to include(winter_fruits) }
    end
  end
end
{% endhighlight %}

#### Example #2 Service Object

A controller for a fruit search
{% highlight ruby %}
class FruitsController < ApplicationController
  def search
    @fruits = ComplexFruitSearch.new.results(params)
  end
end
{% endhighlight %}

First version of the service object
{% highlight ruby %}
class ComplexFruitSearch
  def results(params)
    @params = params

    Fruit.where(orm_friendly_params)
  end

  private

  def orm_friendly_params
    # complex query logic
  end
end
{% endhighlight %}

A more DI version of the service object
{% highlight ruby %}
class ComplexFruitSearch
  def results(params: params, scope: Fruit.all)
    @params = params

    scope.where(orm_friendly_params)
  end

  private

  def orm_friendly_params
    # complex query logic
  end
end
{% endhighlight %}

A new search category gets added, but thanks to DI we can easily leverage the existing service object.
{% highlight ruby %}
class VegetablesSearchController < ApplicationController
  def in_season
    @results = ComplexProduceSearch.new.results(
      scope: Vegetable.in_season,
      params: params
    )
  end

  def pulses
    @results = ComplexProduceSearch.new.results(
      scope: Vegetable.pulses,
      params: params
    )
  end
end
{% endhighlight %}

`ComplexProduceSearch.new` is repeated, so let's remove duplication, and make the service object take advantage of being an instance.

{% highlight ruby %}
class VegetablesSearchController < ApplicationController
  def in_season
    @results = veg_search.call(
      scope: :in_season,
      params: params
    )
  end

  def pulses
    @results = veg_search.call(
      scope: :pulses,
      params: params
    )
  end

  private

  def veg_search
    ComplexProduceSearch.new(type: Vegetable)
  end
end

# DI + Configurable objects = Joy
class ComplexProduceSearch
  def initialize(type:)
    @type = type
  end

  def call(params:, scope:)
    @params = params
    @scope = scope

    type.public_send(scope).where(orm_friendly_params)
  end

  private

  attr_reader :type, :params, :scope

  def orm_friendly_params
    # complex query logic
  end
end
{% endhighlight %}

With these changes the hardcoded dependency has become a runtime dependency. Now we are programming to an
interface instead of an implementation.

#### Ducks and Mallards

{% highlight ruby %}
# ComplexProduceSearch is a Mallard (concrete class)
@results = ComplexProduceSearch.new(type: Fruit).call(
  scope: :in_season,
  params: params
)

# fruit_search is a duck, that only exposes an interface
# makes it easy to change the underlying class to another with the same interface
@results = fruit_search.new(type: Fruit).call(
  scope: :in_season,
  params: params
)
{% endhighlight %}

#### Object construction

Compare
{% highlight ruby %}
# Class method
ComplexProduceSearch.results(Fruit.in_season, params)
{% endhighlight %}
Not OO, more like a namespaced procedure, implementation hard to refactor

{% highlight ruby %}
# Pass everything to new
ComplexProduceSearch.new(Fruit.in_season, params).results
{% endhighlight %}
More or less the same as before, though there is an instance. But this instance isn't reusable.

{% highlight ruby %}
# Separate dependencies and inputs
ComplexProduceSearch.new(Fruit).results(:in_season, params)
{% endhighlight %}

Instantiation and invocation have been separated. The object can reused for different inputs.

You can take this even further in your application by creating instantiation as a separate concern.

{% highlight ruby %}
class VegetableSeachController < ApplicationController
  def in_season
    @results = app.veg_search.call(
      scope: :in_season,
      params: params
    )
  end
end

# Separate business logic app from the framework
class FruitOfTheMonthApp
  def veg_search
    @veg_search ||= ComplexProduceSearch.new(Vegetable)
  end
end
{% endhighlight %}

But how to get it into Rails?  Stephen suggests to add a constant to the initializers

{% highlight ruby %}
# config/initializers/fruit_of_the_month_app.rb
APP = FruitOfTheMonthApp.new

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  def app
    APP
  end
end
{% endhighlight %}

This sounds like a resonable approach, but it breaks [Rails reloading](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html#autoloading-and-initializers).
To make a ruby object a singleton (thanks to our functional approach, we want to use the same instance through
the lifecycle of our application), we can just include the [Singleton
module](http://ruby-doc.org/stdlib-1.9.3/libdoc/singleton/rdoc/Singleton.html).

{% highlight ruby %}
require 'singleton'

class FruitOfTheMonthApp
  include Singleton # voilà!

  def veg_search
    @veg_search ||= ComplexProduceSearch.new(Vegetable)
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  def app
    FruitOfTheMonthApp.instance
  end
end
{% endhighlight %}
Though still leaves us with the dependency to the concrete App class.

#### Functional tricks

Compare

The mallard approach
{% highlight ruby %}
class Createuser
  def call(params)
     user = User.new(params)

     # complex logic
  end
end
{% endhighlight %}

With dependency injection
{% highlight ruby %}
class CreateUser
  def initialize(user_class:)
    @user_class = user_class
  end

  def call(params)
    @user_class.new(params)

    # complex logic
  end
end
{% endhighlight %}
Still coupled with the interface of the user_class

Lets abstract away the interface to the user_class
{% highlight ruby %}
class CreateUser
  def initialize(:user_builder)
    @user_builder = user_builder
  end

  def call(params)
    @user_builder.call(params)

    # complex logic
  end
end
{% endhighlight %}
User_builder can be anything that responds to #call (procs, blocks, lambdas)...

..or for example the #method method
{% highlight ruby %}
CreateUser.new(user_builder: User.method(:new))
{% endhighlight %}
This user_builder can be easily replaced with a more comples object later.

#### Benefits of DI

1. More focused and faster tests
2. More flexible code
3. Code that can do different things when introduced to new collaborators
4. Reduced need to change existing objects
5. A clear path for acheiving the 'O', 'I' and 'D' of SOLID 
