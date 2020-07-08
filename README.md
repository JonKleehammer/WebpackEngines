# WebpackEngines
Repository with instructions and example code for mounting engines and compiling them with Webpacker following what I'll refer to as the [7 Step Guide](https://github.com/rails/webpacker/blob/master/docs/engines.md). I've written this guide to be as detailed as possible as the 7 step guide seemed to assume a lot of background knowledge and I felt left out some valuable steps. The first part is pretty basic and just about creating the app and engines, then hooking them together. For this guide I'm using 2 engines to demonstrate the full capabiltiies all steps are the same between them unless stated otherwise. You can use one engine to get off the ground. If you already know how to create an engine and hook it into an app you can skip to step 2 where we actually start using webpacker.

Disclaimer: Guides linked are pretty general, but they give a decent foundation to work from.

## Step 1: Initializing Parent App and Engines

I'm using Rubymine which has options for initializing a new app. I will provide instructions for using rubymine then I will show the command rubymine generated to initialize the project.

### 1-0: Creating the Parent App [(guide)](https://guides.rubyonrails.org/getting_started.html#creating-the-blog-application)
![Rubymine New Rails App Settings](/images/initParentApp.png)

Which generated the equivalent to:
`rails new ParentApp --webpack=vue --skip`

### 1-1: Generating Engines [(guide)](https://guides.rubyonrails.org/engines.html#generating-an-engine)
I will be using 2 separate engines for this guide (Engine2 has the same settings). One thing to note is that Rubymine provides the option for `--wepack=vue`, but it doesn't actually seem to do anything.

![Rubymine New Rails Engine Settings](/images/initEngine.png)

which generated the equivalent to:
`rails plugin new Engine1 --webpack=vue --mountable --skip`

#### File structure

The 7 step guide has the engines stored within the ParentApp, whereas I have my engines in separate directories, we'll be using the absolute path within the engines or app to reference anything outside itself.

```
WebpackEngines (Repository)
  -ParentApp (The App mounting the engines)
  -Engine1
  -Engine2
```

### 1-2: Filling out gemspec

We have to fill out the require sections of the gemspec. I'm removing everything that isn't required by looking at the [specifications](https://guides.rubygems.org/specification-reference/)
I removed email, homepage, description, and license then filled out 
```
# WebpackEngines/Engine1/engine1.gemspec

Gem::Specification.new do |spec|
  spec.name        = "engine1"
  spec.version     = Engine1::VERSION
  spec.authors     = ["Jon Kleehammer"]
  spec.summary     = "An engine to be mounted by a parent app"
  
  # rest of the gemspec unaltered
```

### 1-3: Generate basic controllers and views for main app and engines [(guide)](https://guides.rubyonrails.org/engines.html#providing-engine-functionality)

Within the parent app and each engine I'm going to generate a controller, with different names to better differentiate them. This will provide us with controllers and views so we can check that everything is working properly later on. 

In ParentApp: `rails g controller Home index`

In Engine1: `rails g controller Welcome index`

In Engine2: `rails g controller Greetings index`

### 1-4: Hooking the engines into the main app [(guide)](https://guides.rubyonrails.org/engines.html#hooking-into-an-application)

Now we hook the engines into the main app by adding them to the gemfile. Add all your engines to the gemfile, because it's a local gem we'll also need provide the path to the gem.

```
# ParentApp/Gemfile

gem 'engine1', path: 'C:\Users\Amber Bamber\Documents\JonStuff\LeaseAnalytics\WebpackEngines\Engine1'
gem 'engine2', path: 'C:\Users\Amber Bamber\Documents\JonStuff\LeaseAnalytics\WebpackEngines\Engine2'

```

**Once the engines are added**, run `bundle install` in the ParentApp.

Now add the routes to the engines

```
# ParentApp/config/routes.rb

Rails.application.routes.draw do
  get 'home/index'

  mount Engine1::Engine, at: '/engine1'
  mount Engine2::Engine, at: '/engine2'
end
```

Once the routes are add you can check that everything is working by running `rails s` in the ParentApp

Going to `localhost:3000` will load up the default page since we didn't set a route for our root
![image](imagePath)

Going to `localhost:3000/home/index` will go to our first view in the ParentApp
![image](imagePath)

**But we'll get an error** if we try going to a view from an engine like `localhost:3000/engine1/welcome/index`
![image](imagePath)

By following the directions on the error it can be easily fixed. Add the following lines in ParentApp
`ParentApp/app/assets/config/manifest.js`
```
// ParentApp/app/assets/config/manifest.js

//= link_tree ../images
//= link_directory ../stylesheets .css

//= link engine1/application.css
//= link engine2/application.css
```

You can now reload the page and see the view as expected.

![image](imagePath)

**The engines are now successfully mounted into the main app**. This was all how I did the first step in the 7 step guide. From here we can follow the guide more closely and I will add any explanation I think is necessary along the way

# Following the [7 Step Guide](https://github.com/rails/webpacker/blob/master/docs/engines.md)

We just completed step 1 which didn't have a lot of explanation in the original guide. Now for the next 6 steps we can follow more closely.

## Step 2: install Webpacker within the engine.

There is no built-in tasks to install Webpacker within the engine, thus you have to add all the require files manually (you can copy them from the main app):
- Add `config/webpacker.yml`
- Add `config/webpack/*.js` *(I just did the entire webpack directory `config/webpack/*`)*
- Add `bin/webpack` 
- Add `bin/webpack-dev-server`
- Add `package.json` with required deps.

*It says to add `package.json` with required dependencies. To do this I ran `npm install` which will install everything listed in package.json*

## Step 3: configure Webpacker instance.
*Make sure to change everywhere it says Engine1 to the name of your engine*

- File `lib/engine1.rb`

```ruby
require "engine1/engine"

module Engine1
  ROOT_PATH = Pathname.new(File.join(__dir__, ".."))

  class << self
    def webpacker
      @webpacker ||= ::Webpacker::Instance.new(
        root_path: ROOT_PATH,
        config_path: ROOT_PATH.join("config/webpacker.yml")
      )
    end
  end
end
```

## Step 4: Configure dev server proxy.
*Make sure to change everywhere it says Engine1 to the name of your engine*

- File `lib/engine1/engine.rb`

```ruby
module Engine1
  class Engine < ::Rails::Engine
    initializer "webpacker.proxy" do |app|
      insert_middleware = begin
                            Engine1.webpacker.config.dev_server.present?
                          rescue
                            nil
                          end
      next unless insert_middleware

      app.middleware.insert_before(
          0, Webpacker::DevServerProxy, # "Webpacker::DevServerProxy" if Rails version < 5
          ssl_verify_none: true,
          webpacker: Engine1.webpacker
      )
    end
  end
end
```

**There is a part of step 4 in the original guide for configurations with multiple webpackers. I think that may be something I should implement.**

## Step 5: configure helper.
*Make sure to change everywhere it says Engine1 to the name of your engine*

- File `app/helpers/engine1/application_helper.rb`

```ruby
require "webpacker/helper"

module Engine1
  module ApplicationHelper
    include ::Webpacker::Helper

    def current_webpacker_instance
      Engine1.webpacker
    end
  end
end
```

Now you can use `stylesheet_pack_tag` and `javascript_pack_tag` from within your engine.

## Step 6: rake tasks.

Add Rake task to compile assets in production (~~`rake my_engine:webpacker:compile`~~) *This is not what the command was for me*

*This task is what will actually compile the assets using webpacker which we'll call when we want to compile. In the next step we'll configure webpack to tell it where to compile to.*

*Be very careful when putting this into your own project, there are 5 places you'll need to change Engine1 to your engine name, don't forget any*

- ~~File `my_engine_rootlib/tasks/my_engine_tasks.rake`~~
- File `Engine1/lib/tasks/engine1_tasks.rake`


```ruby
def ensure_log_goes_to_stdout
  old_logger = Webpacker.logger
  Webpacker.logger = ActiveSupport::Logger.new(STDOUT)
  yield
ensure
  Webpacker.logger = old_logger
end


namespace :engine1 do
  namespace :webpacker do
    desc "Install deps with yarn"
    task :yarn_install do
      Dir.chdir(File.join(__dir__, "../..")) do
        system "yarn install --no-progress --production"
      end
    end

    desc "Compile JavaScript packs using webpack for production with digests"
    task compile: [:yarn_install, :environment] do
      Webpacker.with_node_env("production") do
        ensure_log_goes_to_stdout do
          if Engine1.webpacker.commands.compile
            # Successful compilation!
          else
            # Failed compilation
            exit!
          end
        end
      end
    end
  end
end

def yarn_install_available?
  rails_major = Rails::VERSION::MAJOR
  rails_minor = Rails::VERSION::MINOR

  rails_major > 5 || (rails_major == 5 && rails_minor >= 1)
end

def enhance_assets_precompile
  # yarn:install was added in Rails 5.1
  deps = yarn_install_available? ? [] : ["engine1:webpacker:yarn_install"]
  Rake::Task["assets:precompile"].enhance(deps) do
    Rake::Task["engine1:webpacker:compile"].invoke
  end
end

# Compile packs after we've compiled all other assets during precompilation
skip_webpacker_precompile = %w(no false n f).include?(ENV["WEBPACKER_PRECOMPILE"])

unless skip_webpacker_precompile
  if Rake::Task.task_defined?("assets:precompile")
    enhance_assets_precompile
  else
    Rake::Task.define_task("assets:precompile" => "engine1:webpacker:compile")
  end
end
```

## Step 7: serving compiled packs.

There are two approaches on serving compiled assets. *I am using the first option. If you want to see the second check the original guide.*

### Put engine's assets to the root app's public/ folder

You can serve engine's assets using the main app's static files server which serves files from `public/` folder.

For that you must configure your engine's webpacker to put compiled assets to the app's `public/` folder:
*Currently I recommend creating a new directory to store each engine's pack's. I created: `ParentApp/public/engine1-packs`*

```yml
# Engine1/config/webpacker.yml
default: &default
  # ...
  # public_root_path could be used to override the path to `public/` folder
  # (relative to the engine root)
  public_root_path: C:\Users\Amber Bamber\Documents\JonStuff\LeaseAnalytics\WebpackEngines\ParentApp\public
  # use a different sub-folder name
  public_output_path: engine1-packs
  
  # ...
```



















