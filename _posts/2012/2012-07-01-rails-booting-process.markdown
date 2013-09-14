---
title:  Rails Booting Process v3.2.5
date: 2012-07-01
slug: 2012/07/01/rails-booting-process
layout: post
categories: notes
tags: [ruby rails booting initialize]
desc:  Rails Booting Process v3.2.5  
---
There are serval important gems like __activesuppot__, __activerecord__, __actionpack__...etc in rails. So how do rails initizlize these gems and boot? __Rails::Application__ have an overview about the process:  

> == Booting process ==  
> 1)  require "config/boot.rb" to setup load paths  
> 2)  require railties and engines  
> 3)  Define Rails.application as "class MyApp::Application < Rails::Application"  
> 4)  Run config.before_configuration callbacks  
> 5)  Load config/environments/ENV.rb  
> 6)  Run config.before_initialize callbacks  
> 7)  Run Railtie#initializer defined by railties, engines and application. One by one, each engine sets up its load paths, routes and runs its config/initializers/* files.  
> 9)  Custom Railtie#initializers added by railties, engines and applications are executed  
> 10) Build the middleware stack and run to_prepare callbacks  
> 11) Run config.before_eager_load and eager_load if cache classes is true  
> 12) Run config.after_initialize callbacks  

*where is step 8 ?*  

Type `rails new myapp` new a rails project, then we type `rails server` in terminal to rackup the rails server.  

### bin/rails script
`rails` script will load `railties/bin/rails` script in background, `railties/bin/rails` require 'rails/cli' in railties gem.  

### rails/cli.rb
`rails/cli` firstly will require `rbconfig` and `rails/script_rails_loader`, and then excute __Rails::ScriptRailsLoader.exec_script_rails!__

> RbConfig  #rbconfig.rb is written by mkconfig.rb, (when the Ruby was built,) which may have itself changed over the various versions. Also, on Windows, some users install a one-click installer edition, that was not built on their own computer, and they may install it in a custom path, which may invalidate some of the values in the rbconfig.rb file.

### script/rails
Rails::ScriptRailsLoader.exec_script_rails! check curruent path whether have 'script/rails', if so, exec 'ruby script/rails server'. 

``` ruby
APP_PATH = File.expand_path('../../config/application',  __FILE__)
require File.expand_path('../../config/boot',  __FILE__)
require 'rails/commands'
```

## (1) require "config/boot.rb" to setup load paths  
`config/boot.rb` require 'rubygems' and setup bundler by Gemfile.

### rails/commands.rb
`rails/commands.rb` will take ARGV we passed to excute specify action, support 'generate', 'destroy', 'plugin', 'benchmarker', 'profiler', 'console', 'server', 'dbconsole', 'application', 'runner', 'new', '--version'... Here we pass __server__, codes below will be excuted: 

``` ruby
require 'rails/commands/server'
Rails::Server.new.tap { |server|
  # We need to require application after the server sets environment,
  # otherwise the --environment option given to the server won't propagate.
  require APP_PATH
  Dir.chdir(Rails.application.root)
  server.start
}
```

### rails/commands/server
`rails/commands/server` will require 'action_dispatch', defined __Rails::Server__ which inherited from __Rack::Server__, when Rails::Server.new will will call __Rack::Server.initialize__ method first, set ENV["RAILS_ENV"] and then default options for servers:  

> :environment=>"development",  
> :pid=>"/home/ysorigin/projects/iflyee/myapp/tmp/pids/server.pid",  
> :Port=>3000,  
> :Host=>"0.0.0.0",   
> :AccessLog=>[],  
> :config=>"/home/ysorigin/projects/iflyee/myapp/config.ru",  
> :daemonize=>false,  
> :debugger=>false,  
> :server=>nil  

### action_dispatch

``` ruby
require 'active_support'  
require 'active_support/dependencies/autoload'  
require 'action_pack'  
  # only version loaded
require 'active_model'  
  # active_model will require I18n, and add its locales/en file to I18n.load path  
require 'rack'  
```

After initialize server require __APP_PATH__ set in script/rails which is 'config/application.rb', and chdir to project root and start to run.

## (2)  require railties and engines  

### config/application.rb
__appllication.rb__ require `config/boot.rb` again, and `rails/all`.

``` ruby
##rails/all
require "rails"
%w(
  active_record
  action_controller
  action_mailer
  active_resource
  rails/test_unit
  sprockets
).each do |framework|
  begin
    require "#{framework}/railtie"
```

### rails.rb
__rails.rb__ load __rails/ruby_version_check__, this will check ruby version for Rails 3, requires Ruby 1.8.7 or 1.9.2 above, __1.9.1__ have segfaulted the test suite. and load active_support/railtie and active_dispatch/railtie.

and __config/application.rb__ also define our __Myapp::Application__ < __Rails::Application__.  

Railties will load by this sort: I18n::Railtie, ActiveSupport::Railtie, ActionDispatch::Railtie, ActionView::Railtie, ActionController::Railtie, ActiveRecord::Railtie, ActionMailer::Railtie, ActiveResource::Railtie, Sprockets::Railtie, Jquery::Rails::Engine, Remotipart::Rails::Engine, Sass::Rails::Railtie, Coffee::Rails::Engine, MyApp::Application  

All required railties inherited from __Rails::Railtie__, and its `inherited` will be excuted to include __Railtie::Configurable__ module *railtie.rb # Line 131*.  

## (3) Define Rails.application as __class MyApp::Application__
__MyApp::Application__ < __Rails::Application__ < __Rails::Engine__ < __Rails::Railtie__, afterl our Rails.application defined, __Rails::Application.inherited__ will be excuted:

``` ruby
Rails.application = base.instance   # set Rails.application
#load classes in lib and use them during application configuration.
Rails.application.add_lib_to_load_path!  
# run before_configuration callbacks
ActiveSupport.run_load_hooks(:before_configuration, base.instance)
```

## (4) Run config.before_configuration callbacks  

And then is Rails::Engine.inherited and Rails::Railtie.inherited:  

``` ruby
# Rails::Engine.inherited  
Rails.application = base.instance   # set Rails.application
#load classes in lib and use them during application configuration.
Rails.application.add_lib_to_load_path!  
# run before_configuration callbacks, like sass, haml have this block defined 
ActiveSupport.run_load_hooks(:before_configuration, base.instance)

# Rails::Railtie.inherited
base.send(:include, Railtie::Configurable)
subclasses << base
```

When we use `config` to add options to application, __Rails::Application__ initialize __Rails::Application::Configuration__ which < __Rails::Engine::Configuration__ < __Rails::Railtie::Configuration__.  
After this go back to __rails/commands.rb__, rails server run start method.Here it setup url and puts some logger to terminal and created tmp dirs for cache, pids, sessions, sockets, and then call __super__, which it's __Rack::Server.start__. 

``` ruby
url = "#{options[:SSLEnable] ? 'https' : 'http'}://#{options[:Host]}:#{options[:Port]}"
puts "=> Booting #{ActiveSupport::Inflector.demodulize(server)}"
puts "=> Rails #{Rails.version} application starting in #{Rails.env} on #{url}"
puts "=> Call with -d to detach" unless options[:daemonize]
trap(:INT) { exit }
puts "=> Ctrl-C to shutdown server" unless options[:daemonize]
%w(cache pids sessions sockets).each do |dir_to_make|
  FileUtils.mkdir_p(Rails.root.join('tmp', dir_to_make))
end

super
```

### Rack::Server.start
According default options __Rack::server__ will set varibles and require other libs, and call `wrapped_app` which initizlize app --> Rack::Builder parse __config.ru__ file into Rack Application(excute config.ru) ,and built the app --> call the default middlewares [__Rails::Rack::LogTailer__, __Rack::ContentLength__].

``` ruby
#######config.ru#######
require File.expand_path('../application', __FILE__)
# Initialize the rails application
Myapp::Application.initialize!
```

Here __Myapp::Application.initialize!__ will excute __Rails::Initializable.run_initializers(:default)__ call initializers define in engine.rb, application/boostrap.rb application/finisher.rb and other railties(activerecord, action_controller, action_dispatch...) with sorts. `initializer(name, opts, &blk)`, opts set as __{:before => :xxx}__ or __{:after => :xx }__ to pass so that we can specify :xxx before or after some initializer to excute.
After initializers get sort, will be excuted by this sort: 

#### * #set_load_path
Add configured load paths to ruby load paths and remove duplicates.  

#### * #set_autoload_paths
Add load paths to ruby to load, $LOAD_PATH, and uniq remove duplicate paths,Set the paths from which Rails will automatically load source files, and the load_once paths. *config* autoload_paths, eager_loaded_path.

#### * #add_routing_paths
Load config/routes.rb app.routes_reloader.route_sets << routes

#### * #add_locales
Add config/locales to i18n #load i18n locales  ##added later, but load higher priority

#### * #add_view_paths
ActiveSupport.on_load views path

## (5)  Load config/environments/ENV.rb  

#### * #load_environment_config

``` ruby
@_env ||= ActiveSupport::StringInquirer.new(ENV["RAILS_ENV"] || ENV["RACK_ENV"] || "development"),
```

paths will add config/environments/xxx.rb according __Rails.env__, as __@_env__. ENV is Ruby hash-like accessor for environment variables.
by default it will load config/development.rb 

#### * #load_environment_hook

#### * #load_active_support
require "active_support/all"

#### * #preload_frameworks
Preload all frameworks specified by the rails.config#frameworks.

#### * #initialize_logger
Initialize logger, Rails.logger || config.logger || ActiveSupport::TaggedLogging

#### * #initialize_cache
Set __Rails.cache__ to ActiveSupport::Cache.new.

#### * #initialize_dependency_mechanism
Sets the dependency loading mechanism.## ActiveSupport::Dependencies.mechanism = config.cache_classes ? :require : :load

## (6)  Run config.before_initialize callbacks  

#### * #bootstrap_hook
ActiveSupport.run_load_hooks(:before_initialize, app)

## (7)  Run Railtie#initializer defined by railties, engines and application. One by one, each engine sets up its load paths, routes and runs its config/initializers/* files.  

#### * #i18n.callbacks
add I18n.reloader(ActiveSupport::FileUpdateChecker checking locales file, it provides API for Rails to watch files and control reloading) to ActionDispatch::Reloader middleware __to_prepare!__ callback(like before filter).By default ActionDispatch::Reloader is included in the middleware stack only in the development environment.

#### * #active_support.initialize_whiny_nils
__require 'active_support/whiny_nil'__, whiny_nil extend __NilClass__ id method, when call nil.id, it will raise RunTimeError.

#### * #active_support.deprecation_behavior
Setup where active_support deprecation messages to print, default "development" => :log, "production"  => :notify, "test" => :stderr

#### * #active_support.initialize_time_zone
Setup TimeZone with config.time_zone value.

#### * #action_dispatch.configure

``` ruby
ActionDispatch::Http::URL.tld_length = app.config.action_dispatch.tld_length
  # [TLD(top level domain)](http://en.wikipedia.org/wiki/Top-level_domain), default is 1
  # ActionDispatch::Http::URL.extract_domain("www.google.com", tld_length=1)  # google.com
  # ActionDispatch::Http::URL.extract_domain("www.google.com", tle_length=2)  #www.google.com

ActionDispatch::Request.ignore_accept_header = app.config.action_dispatch.ignore_accept_header
  # if request didn't set params[:format], will check ignore_accept_header will returns the accepted MIME type for the request, which is "action_dispatch.request.formats"

ActionDispatch::Response.default_charset = app.config.action_dispatch.default_charset || app.config.encoding
  # response default charset

ActionDispatch::ExceptionWrapper.rescue_responses.merge!(config.action_dispatch.rescue_responses)
ActionDispatch::ExceptionWrapper.rescue_templates.merge!(config.action_dispatch.rescue_templates)
  # add more rescue error and templates for exception raise and show it then

config.action_dispatch.always_write_cookie = Rails.env.development? if config.action_dispatch.always_write_cookie.nil?
ActionDispatch::Cookies::CookieJar.always_write_cookie = config.action_dispatch.always_write_cookie
  # whether to set always for cookies, for development default is true
```

#### * #action_view.embed_authenticity_token_in_remote_forms
   ActionView::Helpers::FormTagHelper.embed_authenticity_token_in_remote_forms =
      app.config.action_view.delete(:embed_authenticity_token_in_remote_forms)
      #By default it's true, embeded authenticity_token for the form

#### * #action_view.cache_asset_ids
   ActionView::Helpers::AssetTagHelper::AssetPaths.cache_asset_ids = false
      # With the cache enabled, the asset tag helper methods will make fewer expensive file system calls (the default implementation checks the file system timestamp). However this prevents you from modifying any asset files while the server is running.

#### * #action_view.javascript_expansions [should be change name to javascript_and_stylesheet?]

``` ruby
ActionView::Helpers::AssetTagHelper.register_javascript_expansion(app.config.action_view.delete(:javascript_expansions))
# Register one or more javascript files to be included when symbol
# is passed to javascript_include_tag. This method is typically intended
# to be called from plugin initialization to register javascript files
# that the plugin installed in public/javascripts.
#
#   ActionView::Helpers::AssetTagHelper.register_javascript_expansion :monkey => ["head", "body", "tail"]
#
#   javascript_include_tag :monkey # =>
#     <script type="text/javascript" src="/javascripts/head.js"></script>
#     <script type="text/javascript" src="/javascripts/body.js"></script>
#     <script type="text/javascript" src="/javascripts/tail.js"></script>
ActionView::Helpers::AssetTagHelper.register_stylesheet_expansion(app.config.action_view.delete(:stylesheet_expansions))
```

# more or less with register_javascript_expansion

#### * #action_view.set_configs
app.config.action_view.each {|k,v| send "#{k}=", v}, initialize config variables 
    
#### * #action_view.caching
 
``` ruby
if app.config.action_view.cache_template_loading.nil?
  ActionView::Resolver.caching = app.config.cache_classes
end
#cache view templates, default is true
```

#### * #action_controller.logger
Set action_controller.logger to Rails.logger

#### * #action_controller.initialize_framework_caches
Set action_controller cache to default rails cache, initizlize in railtie 'initialize_cache' initializer. Default it's 
:file_store. check active_support more details.

#### * #action_controller.assets_config
Set action_controller.assets_dir to 'public' dir

#### * #action_controller.set_configs

``` ruby
javascripts_dir      ||= paths["public/javascripts"].first
stylesheets_dir      ||= paths["public/stylesheets"].first
page_cache_directory ||= paths["public"].first
                                                           
ure readers methods get compiled
asset_path           ||= app.config.asset_path#### #action_controller.compile_config_methods
asset_host           ||= app.config.asset_host
relative_url_root    ||= app.config.relative_url_root#### #active_record.initialize_timezone
#set these config to action_controller as method
```

#### * #active_record.logger
Set active_record.logger to Rails.logger

#### * #active_record.identity_map
Insert __ActiveRecord::IdentityMap::Middleware__ middleware after __ActionDispatch::Callbacks__ middleware to config.middlewares, default it's nil,set up mannual

#### * #active_record.set_configs
Delete whitelist attributes value, and

``` ruby
app.config.active_record.each do |k,v|
  send "#{k}=", v
end
```

#### * #active_record.initialize_database
Establish connection to databse through config/database.yml file

#### * #active_record.log_runtime
Include ActiveRecord::Railties::ControllerRuntime, logger active_record run time

#### * #active_record.set_reloader_hooks
According __config.reload_classes_only_on_change__(default is true), set up the callback __ActiveRecord::Base.clear_reloadable_connections!__ and __ActiveRecord::Base.clear_cache!__ to excuted by ActionDispatch prepare or cleanup callbacks.

#### * #active_record.add_watchable_files
config.watchable_files.concat ["#{app.root}/db/schema.rb", "#{app.root}/db/structure.sql"]

#### * #action_mailer.logger
Set active_mailer.logger to Rails.logger

#### * #action_mailer.set_configs

``` ruby
javascripts_dir      ||= paths["public/javascripts"].first
stylesheets_dir      ||= paths["public/stylesheets"].first
page_cache_directory ||= paths["public"].first
                                                           
ure readers methods get compiled
asset_path           ||= app.config.asset_path#### #action_controller.compile_config_methods
asset_host           ||= app.config.asset_host
relative_url_root    ||= app.config.relative_url_root#### #active_record.initialize_timezone
# set these config to action_mailer as method
register_interceptors(options.delete(:interceptors))
# Register one or more Interceptors which will be called before mail is sent.

register_observers(options.delete(:observers))
# Register one or more Observers which will be notified when mail is delivered.
```

#### * #action_mailer.compile_config_methods

``` ruby
config.compile_methods! if config.respond_to?(:compile_methods!), for now active_support configuration responsed.
```

#### * #active_resource.set_configs

``` ruby
app.config.active_resource.each do |k,v|
  ActiveResource::Base.send "#{k}=", v
end
```

#### * #sprockets.environment
  setup sprockets environment, logger, cache, load manifest.yml and load helpers.

#### * #setup_sass
Override stylesheet engine to the preferred syntax  
Set the sass cache location  
Establish configuration defaults that are evironmental in nature  

#### * #setup_compression

``` ruby
app.config.sass.style = :compressed
app.config.assets.css_compressor = CssCompressor.new
```

#### * #append_assets_path

``` ruby
app.config.assets.paths.unshift(*paths["vendor/assets"].existent_directories)
app.config.assets.paths.unshift(*paths["lib/assets"].existent_directories)
app.config.assets.paths.unshift(*paths["app/assets"].existent_directories)
```

#### * #prepend_helpers_path
app.config.helpers_paths.unshift(*paths["app/helpers"].existent)

## (9)  Custom Railtie#initializers added by railties, engines and applications are executed  

#### * #load_config_initializers
Load all config/initializers, including other gems, plugins.

#### * #add_generator_templates

#### * #ensure_autoload_once_paths_as_subset

#### * #add_builtin_route
For development ##match '/rails/info/properties' => "rails/info#properties" 

## (10) Build the middleware stack and run to_prepare callbacks  
#### * #build_middleware_stack
Include default middlewares 

#### * #define_main_app_helper
app.routes.define_mounted_helper(:main_app)

#### * #add_to_prepare_blocks
ActionDispatch::Reloader.to_prepare(blk)

#### * #run_prepare_callbacks
ActionDispatch::Reloader.prepare!

## (11) Run config.before_eager_load and eager_load if cache classes is true  

#### * #eager_load!
ActiveSupport.run_load_hooks(:before_eager_load, self)
railties.all(&:eager_load!), excute eager_load! for all railties

## (12) Run config.after_initialize callbacks  

#### * #finisher_hook
ActiveSupport.run_load_hooks(:after_initialize, self)

#### * #set_routes_reloader_hook
Set app reload just after the finisher hook to ensure routes added in the hook are still loaded.

#### * #set_clear_dependencies_hook
Set app reload just after the finisher hook to ensure paths added in the hook are still loaded.

#### * #disable_dependency_loading
ActiveSupport::Dependencies.unhook!
'
Finally __Rack::Server__ pass the app to handler(webrick, thin or others) to receive request from browsers.Rails Booting done! ![](/assets/images/emotions/00.jpg)
