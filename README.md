# Dependor 

[![build status](https://secure.travis-ci.org/psyho/dependor.png)](http://travis-ci.org/psyho/dependor)
[![Code Climate](https://codeclimate.com/badge.png)](https://codeclimate.com/github/psyho/dependor)

## What is Dependor

Dependor is a set of helpers that make writing Ruby apps that use the dependency injection pattern easier.
It comes as a set of modules, which you can selectively add to your project.
It is designed do play nice with Rails and similar frameworks.

## Manual Dependency Injection

```ruby
class Foo
  def do_foo
    "foo"
  end
end

class Bar
  def initialize(foo)
    @foo = foo
  end

  def do_bar
    @foo.do_foo + "bar"
  end
end

class Injector
  def foo
    Foo.new
  end

  def bar
    Bar.new(foo)
  end
end

class EntryPoint
  def inject
    @injector ||= Injector.new
  end

  def bar
    inject.bar
  end

  def run
    bar.do_bar
  end
end

EntryPoint.new.run
```

## The same thing with Dependor

```ruby
require 'dependor'
require 'dependor/shorty'

class Foo
  def do_foo
    "foo"
  end
end

class Bar
  takes :foo

  def do_bar
    @foo.do_foo + "bar"
  end
end

class Injector
  include Dependor::AutoInject
end

class EntryPoint
  include Dependor::Injectable
  inject_from Injector
  
  inject :bar

  def run
    bar.do_bar
  end
end

EntryPoint.new.run
```

## Dependor::AutoInject

This is the core part of the library.
It looks at the constructor of a class to find out it's dependencies and instantiates it's instances with proper objects injected.
It looks up classes by name.

AutoInject can also use the methods declared on injector as injection sources, which is quite useful for things like configuration.

```ruby
class Injector
  include Dependor::AutoInject

  attr_reader :session

  def initialize(session)
    @session = session
  end

  let(:current_user) { current_user_service.get }
  let(:users_repository) { User }
  let(:comments_repository) { Comment }
end

class CurrentUserService
  takes :session, :users_repository

  def get
    @current_user ||= users_repository.find(session[:current_user_id])
  end
end

class CreatesComments
  takes :current_user, :comments_repository

  def create
    # ...
  end
end

class User < ActiveRecord::Base
end

class Comment < ActiveRecord::Base
end
```

## Dependor::Shorty

This makes the constructor definition less verbose and includes Dependor::Let for shorter method definition syntax.

```ruby
class Foo
  takes :foo, :bar, :baz
  let(:hello) { "world" }
end
```

is equivalent to:

```ruby
class Foo
  attr_reader :foo, :bar, :baz

  def initialize(foo, bar, baz)
    @foo = foo
    @bar = bar
    @baz = baz
  end

  def hello
    "world"
  end
end
```

## Dependor::Constructor

Sometimes you don't want to pollute every class with a `takes` method.
You can then shorten the class declaration with Dependor::Constructor.

```ruby
class Foo
  include Dependor::Constructor(:foo, :bar, :baz)
end
```

is equivalent to:

```ruby
class Foo
  def initialize(foo, bar, baz)
    @foo = foo
    @bar = bar
    @baz = baz
  end
end
```

## Dependor::Let

It allows a simpler syntax to define getter methods.

```ruby
class Foo
  def foo
    do_something_or_other
  end
end
```

becomes:

```ruby
class Foo
  extend Dependor::Let
  let(:foo) { do_something_or_other }
end
```

## Dependor::Injectable

You can include this to make usage of the injector more convenient.
This is used in the entry point of your application, typically a Rails controller.

```ruby
class MyInjector
  def foo
    "foo"
  end
end

class ApplicationController
  extend Dependor::Injectable
  inject_from MyInjector
end

class PostsController < ApplicationController
  inject :foo

  def get
    render text: foo
  end
end
```

Sometimes you might want to pass request, params or session to your injector.
Here is an example, how to do it:

```ruby
require 'dependor/shorty'

class MyInjector
  include Dependor::AutoInject
  
  takes :params, :session, :request

  def foo
    session[:foo]
  end
end

class ApplicationController
  extend Dependor::Injectable

  def injector
    @injector ||= MyInjector.new(params, session, request)
  end
end

class PostsController < ApplicationController
  inject :foo

  def get
    render text: foo
  end
end
```

## License

MIT. See the MIT-LICENSE file.

## Author

Adam Pohorecki

## Acknowledgements

Dependor::Shorty is inspired (or rather blatantly copied) from Gary Bernhardt's [Destroy All Software Screencast][das] ["Shorter Class Syntax"][shorter-syntax].

[das]: http://www.destroyallsoftware.com
[shorter-syntax]: https://www.destroyallsoftware.com/screencasts/catalog/shorter-class-syntax
