!SLIDE first

# _Easy building of ruby web frameworks with Rack_

<br/>
<br/>

### Marcin Kulik

<br/>

EuRuKo,  2010/05/30

!SLIDE big

## About me

 * Senior developer @ Lunar Logic Polska - agile Ruby on Rails development services
 * Working with web for 10 years
 * Using Ruby, Python, Java and others - in love with Ruby since 2007
 * Contributing to Open Source:
   * Open File Fast - quickly open files in Netbeans and JEdit
   * racksh - console for Rack apps
   * Kodr - programmers editor
   * .....


!SLIDE big

## Presentation plan

 * Why would you need another framework
 * How does typical web framework look like
 * Available Rack middleware and tools we can use
 * Q&A

!SLIDE

## Why would you need another framework?

"Because world needs yet another framework ;-)" - Tomash

!SLIDE

## No, you probably don't need it actually :)

 * Tens of mature frameworks
 * Hundreds of custom/experimental ones
 * "Don't reinvent the wheel", right?
 * But...

!SLIDE

## But it's so easy that you should at least try

 * Rack provides everything you'll need, is extremely simple but extremely powerful
 * It will help you to better understand HTTP
 * It will make you better developer
 * Your custom framework will be the fastest one *
 * It's fun! A lot of fun :)

!SLIDE

## Simplest Rack application

@@@ ruby
    run lambda do |env|
      [200, { "Content-type" => "text/plain" }, ["Hello EuRuKo 2010!"]]
    end
@@@

!SLIDE

# Let's transform it into framework!

!SLIDE

## How does typical web framework look like?

  * Rails
  * Merb
  * Pylons
  * Django
  * Rango

!SLIDE

# Looks like MVC, more or less

!SLIDE

## What features we'd like to have?

 * dependencies management
 * RESTful routing
 * controllers
   * session
   * flash messages
 * views
   * layouts
   * templates
   * partials
 * authentication
 * testing
 * console

!SLIDE

# Available Rack middleware and tools we can use

!SLIDE

# (1/7) Gem dependency management

!SLIDE

## bundler

_"A gem to bundle gems"_

[github.com/carlhuda/bundler](http://github.com/carlhuda/bundler)

!SLIDE big

@@@ ruby
    # Gemfile

    source "http://gemcutter.org"
    gem "rack"

    # config.ru

    require "bundler"
    Bundler.setup
    Bundler.require
@@@

!SLIDE

# (2/7) Routing

!SLIDE

## Usher

_"Pure ruby general purpose router with interfaces for rails, rack, email or choose your own adventure"_

[github.com/joshbuddy/usher](http://github.com/joshbuddy/usher)

!SLIDE big

@@@ ruby
    # Gemfile
    
    gem "usher"
    
    # config.ru
    
    require APP_ROOT / "config" / "router.rb"

    run Foobar::Router
@@@

!SLIDE medium

@@@ ruby
    # config/router.rb

    module Foobar
      Router = Usher::Interface.for(:rack) do
        get('/').to(HomeController.action(:welcome)).name(:root) # root URL
        add('/login').to(SessionController.action(:login)).name(:login) # login
        get('/logout').to(SessionController.action(:logout)).name(:logout) # logout
        ...
        default ExceptionsController.action(:not_found) # 404
      end
    end
@@@

!SLIDE

# (3/7) Controller

!SLIDE

## Let's build our base controller

 * every action is valid Rack endpoint
 * value returned from action becomes body of the response

!SLIDE medium

@@@ ruby
    # lib/base_controller.rb
    
    module Foobar
      class BaseController
        def call(env)
          @request = Rack::Request.new(env)
          @response = Rack::Response.new
          resp_text = self.send(env['x-rack.action-name'])
          @response.write(resp_text)
          @response.finish
        end

        def self.action(name)
          lambda do |env|
            env['x-rack.action-name'] = name
            self.new.call(env)
          end
        end
      end
    end
@@@

!SLIDE big

@@@ ruby
    # config.ru
    
    require APP_ROOT / "lib" / "base_controller.rb"
    Dir[APP_ROOT / "app" / "controllers" / "*.rb"].each do |f|
      require f
    end
@@@

!SLIDE

## Now we can create UsersController

@@@ ruby
    # app/controllers/users_controller.rb

    class UsersController < Foobar::BaseController
      def index
        "Hello there!"
      end
    end
@@@

!SLIDE

## Controllers also need following:

 * session access
 * setting flash messages
 * setting HTTP headers
 * redirects
 * url generation
 
!SLIDE

## rack-contrib

_"Contributed Rack Middleware and Utilities"_

[github.com/rack/rack-contrib](http://github.com/rack/rack-contrib)

## rack-flash

_"Simple flash hash implementation for Rack apps"_

[nakajima.github.com/rack-flash](http://nakajima.github.com/rack-flash/)

!SLIDE big

@@@ ruby
    # Gemfile
    
    gem "rack-flash"
    gem "rack-contrib", :require => 'rack/contrib'
    
    # config.ru
    
    use Rack::Flash
    use Rack::Session::Cookie
    use Rack::MethodOverride
    use Rack::NestedParams
@@@

!SLIDE medium

@@@ ruby
    # lib/base_controller.rb
    
    module Foobar
      class BaseController
        def status=(code); @response.status = code; end
        
        def headers; @response.header; end
        
        def session; @request.env['rack.session']; end
        
        def flash; @request.env['x-rack.flash']; end
    
        def url(name, opts={}); Router.generate(name, opts); end

        def redirect_to(url)
          self.status = 302
          headers["Location"] = url
          "You're being redirected"
        end
      end
    end
@@@

!SLIDE medium

## Now we can use #session, #flash and #redirect_to

@@@ ruby
    # app/controllers/users_controller.rb
    
    class UsersController < Foobar::BaseController
      def openid
        if session["openid.url"]
          flash[:notice] = "Cool!"
          redirect_to "/cool"
        else
          render
        end
      end
    end
@@@

!SLIDE

# (4/7) Views

!SLIDE

## Tilt

_"Generic interface to multiple Ruby template engines"_

[github.com/rtomayko/tilt](http://github.com/rtomayko/tilt)

!SLIDE big

@@@ ruby
    # Gemfile
    
    gem "tilt"
@@@

!SLIDE medium

@@@ ruby
    # lib/base_controller.rb
    
    module Foobar
      class BaseController
        def render(template=nil)
          template ||= @request.env['x-rack.action-name']
          template_path = "#{APP_ROOT}/app/views/#{self.class.to_s.underscore}/#{template}.html.erb"
          layout_path = "#{APP_ROOT}/app/views/layouts/application.html.erb"
          Tilt.new(layout_path).render(self) do
            Tilt.new(template_path).render(self)
          end
        end
      end
    end
@@@

!SLIDE

# (5/7) Authentication

!SLIDE

## Warden

_"General Rack Authentication Framework"_

[github.com/hassox/warden](http://github.com/hassox/warden)

!SLIDE medium

@@@ ruby
    # Gemfile
    
    gem "warden"
    
    # config.ru
    
    use Warden::Manager do |manager|
      manager.default_strategies :password
      manager.failure_app = ExceptionsController.action(:unauthenticated)
    end

    require "#{APP_ROOT}/lib/warden.rb"
@@@

!SLIDE medium

@@@ ruby
    # lib/warden.rb
    
    Warden::Manager.serialize_into_session { |user| user.id }
    Warden::Manager.serialize_from_session { |key| User.get(key) }
    
    Warden::Strategies.add(:password) do
      def authenticate!
        u = User.authenticate(params["username"], params["password"])
        u.nil? ? fail!("Could not log in") : success!(u)
      end
    end
@@@

!SLIDE medium

@@@ ruby
    # lib/base_controller.rb
    
    module Foobar
      class BaseController
        def authenticate!
          @request.env['warden'].authenticate!
        end
        
        def logout!(scope=nil)
          @request.env['warden'].logout(scope)
        end
        
        def current_user
          @request.env['warden'].user
        end
      end
    end
@@@

!SLIDE medium

## Now we can guard our action:

@@@ ruby
    # app/controllers/users_controller.rb
    
    class UsersController < Foobar::BaseController
      def index
        authenticate!
        @users = User.all(:id.not => current_user.id)
        render
      end
    end
@@@

!SLIDE

# (6/7) Testing

!SLIDE

## rack-test

_"**Rack::Test** is a small, simple testing API for Rack apps. It can be used on its own or as a reusable starting point for Web frameworks and testing libraries to build on."_

[github.com/brynary/rack-test](http://github.com/brynary/rack-test)

!SLIDE

# (7/7) Console

!SLIDE

## racksh (aka Rack::Shell)

_"**racksh** is a console for Rack based ruby web applications. It's like Rails _script/console_ or Merb's _merb -i_, but for any app built on Rack"_

[github.com/sickill/racksh](http://github.com/sickill/racksh)

!SLIDE medium

## Installation

@@@
    gem install racksh
@@@

## Example racksh session

@@@
    % racksh
    Rack::Shell v0.9.7 started in development environment.
    >> $rack.get "/"
    => #<Rack::MockResponse:0xb68fa7bc @body="<html>...", @headers={"Content-Type"=>"text/html", "Content-Length"=>"1812"}, @status=200, ...
    >> User.count
    => 123
@@@

!SLIDE last

# That's it!

Slides available at: [github.com/sickill/euruko2010-building-frameworks-with-rack](http://github.com/sickill/euruko2010-building-frameworks-with-rack)

Example code available at: [github.com/sickill/example-rack-framework](http://github.com/sickill/example-rack-framework)

Contact me: [marcin.kulik@gmail.com](marcin.kulik@gmail.com)

# Questions?

