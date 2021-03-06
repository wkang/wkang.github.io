---
title: Rails request handle process v3.2.5
date: 2012-07-01
slug: 2012/07/01/rails-request-handle-process-v3.2.5
layout: post
categories: notes
tags: [ruby rails request routes initialize]
desc: The process of rails handle request v3.2.5  
---
When rails booting, initializer `add_routing_paths` will initialize __Myapp::Application.routes__ as `ActionDispatch::Routing::RouteSet.new` which will eval the block defined in __config/routes.rb__. And Myapp::Application.routes will be pushed into rails default middlewares statck as `endpoint`. Defaults rails rack app + default middlewares + endpoint will be ran one by one when webserver service the request.  

Let's check the request flow(default webrick server):
Webrick processes the request on sock, new request and response and then pass to rack webrick handler `service` method.The service method will meta vars in request, and update rack info and HTTP related params to env var, at last call the first rails rack app `LogTailer` to run.  

### Rails::Rack::LogTailer 
Print the logger message to `{Rails.root}/log/{Rails.env}.log` file, will call the next app first, then flush into the file.

### Rails::Rack::Debugger (if pass -d params when excute rails server commands) 
Start the ruby-debugger.

### Rack::ContentLength 
Check the the next middleware response if its header including `Content-Length` and `Transfer-Encoding`, if not set the `Content-Length` with body bytesize for headers.

### ActionDispatch::Static
If the request_method is __GET__ or __HEAD__, check the public dir whether include __path/{,.html,/index.html}__(.html ext from ActionController::Base.page_cache_extension)

### Rack::Lock
__Mutex__ implements a simple semaphore that can be used to coordinate access to shared data from multiple concurrent threads. __Rack::Lock__ set *rack.multithread* to false, and lock the request in the current thread using mutex during the app.call, and then BodyProxy wrapped the response with block that unlock the request. The block will call when the response call close method.

### ActiveSupport::Cache::Strategy::LocalCache
This middleware initialize in the __:initialize_cache__ initializer of __application/bootstrap.rb__, insert before __Rack::Runtime__, configura in __application/configuration.rb__  

> @cache_store  = [ :file_store, "#{root}/tmp/cache/" ]

This wraps up local storage for middlewares, new thread for the cache localstore with thread id as key.  

### Rack::Runtime
Sets an "X-Runtime" response header, indicating the response time of the request, in seconds. Time.now - start_time.  

### Rack::MethodOverride
Override the env["REQUEST_METHOD"] method if set `METHOD_OVERRIDE_PARAM_KEY` and `env[HTTP_METHOD_OVERRIDE_HEADER]` when it's __POST__. Will change `env["rack.methodoverride.original_method"]` and `env["REQUEST_METHOD"]`.  

### ActionDispatch::RequestId
Makes a unique request id available to the action_dispatch.request_id env variable (which is then accessible through ActionDispatch::Request#uuid) and sends the same id to the client via the X-Request-Id header. The unique request id is either based off the X-Request-Id header in the request, which would typically be generated by a firewall, load balancer, or the web server, or, if this header is not available, a random uuid. If the header is accepted from the outside world, we sanitize it to a max of 255 chars and alphanumeric and dashes only. The unique request id can be used to trace a request end-to-end and would typically end up being part of log files from multiple pieces of the stack.

> SecureRandom.hex(16) #generates a random hex string

### Rails::Rack::Logger
Added the request info(path, subdomain, uuid, method, time) to Rails.logger.

### ActionDispatch::ShowExceptions
This middleware rescues any exception returned by the application and calls an exceptions app that will wrap it in a format for the end user.  
Everytime there is an exception, ShowExceptions will store the exception in env["action_dispatch.exception"], rewrite the PATH_INFO to the exception status code and call the rack app.  If the application returns a "X-Cascade" pass response, this middleware will send an empty response as result with the correct status code. If any exception happens inside the exceptions app, this middleware catches the exceptions and returns a FAILSAFE_RESPONSE.

### ActionDispatch::DebugExceptions
This middleware is responsible for logging exceptions and showing a debugging page in case the request is local

### ActionDispatch::RemoteIp
IP addresses that are "trusted proxies" that can be stripped from the comma-delimited list in the X-Forwarded-For header. See also: http://en.wikipedia.org/wiki/Private_network#Private_IPv4_address_spaces

### ActionDispatch::Reloader
__ActionDispatch::Reloader__ provides prepare and cleanup callbacks, intended to assist with code reloading during development.
Prepare callbacks are run before each request, and cleanup callbacks after each request. In this respect they are analogs of __ActionDispatch::Callback__'s before and after callbacks. However, cleanup callbacks are not called until the request is fully complete -- that is, after #close has been called on the response body.(BodyProxy always use to handle body process.)

### ActiveRecord::QueryCache
Enable ActiveRecord query cache sql.

### ActionDispatch::Cookies 
Cookies are read and written through __ActionController#cookies__.The middleware will set or delete the cookies to browsers.

### ActionDispatch::Session::CookieStore
This cookie-based session store is the Rails default. Sessions typically contain at most a user_id and flash message; both fit within the 4K cookie size limit. Cookie-based sessions are dramatically faster than the alternatives.

### ActionDispatch::Flash
The flash provides a way to pass temporary objects between actions. Anything you place in the flash will be exposed to the very next action and then cleared out.Implemented by session.  

### ActionDispatch::ParamsParser
According request.content_mime_type to parse request.body, and then set to env["action_dispatch.request.request_parameters"].

### ActionDispatch::Head
If __env["REQUEST_METHOD"]__ is "HEAD", set to "GET" and __env["rack.methodoverride.original_method"]__ to "HEAD".

### Rack::ConditionalGet 
Middleware that enables conditional GET using If-None-Match and If-Modified-Since. The application should set either or both of the Last-Modified or Etag response headers according to RFC 2616. When either of the conditions is met, the response body is set to be zero length and the response status is set to 304 Not Modified.

### Rack::ETag
Automatically sets the ETag header on all String bodies.

### ActionDispatch::BestStandardsSupport
指定IE8浏览器去模拟某个特定版本的IE浏览器的渲染方式.

### RouteSet
Excute __Journey::Router.call__, get the PATH_INFO from env -> match the route defined -> pass the dispatch app (controller defined in routes.rb or another rack app) to excute call method.

## ActionController
#### ActionController::Base < Metal < AbstractController::Base
__AbstractController::Base__ holds attrs for subclasses(attr_internal,activesupport implement to support module defind _xxx instance variable for class/module) and actions of controller handle related methods.  

__ActionController::Metal__ adds more http handle helper methods like status setter, params, location. Most important is metal will build a middleware stack, so that we can use other middlewares in controller. Metal will process with dispatch method.  

__ActionController::Base__ are the core of a web request in Rails, it includes a bunch of modules: layouts, cookies, route helpers, flash, redirect, render.....etc  

After the route match PATH_INFO, controller will be passed to excute(*actionpack-3.2.5/lib/action_dispatch/routing/route_set.rb:73*) metal `call` class method. Firstly it will find the action with the given env's __action_dispatch.request.path_parameters__ key and pass to to build a middleware stack and then return a block will initialize the metal and call dispatch method. This will invoke rack_delegation first, which will new a response and deleagte __headers__ like methods to _request instance viriable.

`dispatch` initialize the request and env internal attrs, then call `process` action, finally return __[status, headers, response_body]__. Two `process` action will be ran: __abstract_controller/rendering#process__: setup I18n proxy and __abstract_contrller/base__: calls the action going through the entire action dispatch stack. The entry is `process_action` method.Some of modules included in __ActionController::Base__ define `process_action` method, so these `process_action` actions will be also invoked.  

### active_record/railties/controller_runtime
Reset the runtime before each action because of queries in middleware or in cases we are streaming and it won't be cleaned up by the method below.  
ActiveRecord::LogSubscriber.reset_runtime

### action_controller/metal/params_wrapper
Performs parameters wrapping upon the request. Will be called automatically by the metal call stack.

### action_controller/metal/instrumentation
Using ActiveSupport::Notification.instrument to show show related runtime infomation. 

### action_controller/metal/rescue
Provide controllers and configure when detailed exceptions must be shown.

### abstract_controller/callbacks
Run the callbacks around the normal behavior.

### action_controller/metal/rendering
Set the request formats in current controller formats.

### abstract_controller/base
Call the actuall controll action which set in route.  

### action_controller/metal/implicit_render [process_action]
If action not call render, excute the default_render

## render

### action_controller/metal/instrumentation [render]
Benchmark.ms calculate the action_view runtime

### /action_controller/metal/rendering [render]
Setup the conten_type and return response_body

### /abstract_controller/metal/rendering [render]
Through the render params to detemine which view engine and partials should be call.
