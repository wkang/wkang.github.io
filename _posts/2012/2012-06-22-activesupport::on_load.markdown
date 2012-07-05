---
title: ActiveSupport.on_load 
date: 2012-06-22
slug: 2012/06/22/activesupport-on_load
layout: post
categories: notes
tags: [ruby activesupport lazy_load]
desc:  ActiveSupport.on_load lazy load hooks
---
Rails initializer has many ActiveSupport.on_load calls.This method defined in activesupport/lazy_load_hooks.rb.  
ActiveSupport Documents:  

> Lazy load hooks allows rails to lazily load a lot of components and thus making the app boot faster.

## How can it do that?  
Let's see the example in activesupport document, `initializer` provided by Rails::Initializer module will register the block with the name of "active_record.initialize_timezone", when Rails server boots with config.ru, it will require config/environment.rb file, then the environment.rb file will run MyApp::Application.initialize!, in the background Rails::Initializer(rails/initializer.rb) will run block of all initializers registered by `initializer` method.

{:lang='ruby'}
	initializer "active_record.initialize_timezone" do
	 ActiveSupport.on_load(:active_record) do
	   self.time_zone_aware_attributes = true
	   self.default_timezone = :utc
	 end
	end
  
Internal, ActiveSupport module holds two hashes  `@load_hooks = Hash.new { |h,k| h[k] = [] }` and `@loaded = Hash.new { |h,k| h[k] = [] }`, ActiveSuppot.on_load first checks `@loaded` hash whether it holds values of :active_record key.If it did, the block will call; if not, it will insert to `@load_hooks` for excuted later.

*activerecord-3.2.5/lib/active_record/base.rb #Line 721 #activerecord 3.2.5*    

{:lang='ruby'}
	ActiveSupport.run_load_hooks(:active_record, ActiveRecord::Base)

When base.rb file has been evaluated(cause this code at the bottom of the file), ActiveSupport.run_load_hooks will be called.It will add :active_record as key for `@loaded`, and `ActiveRecord::Base` would be inserted to the array.Then loads :active_record hooks defined above code to excute the block.  

###Why we passed ActiveRecord::Base as second params to 'run_load_hooks' method?
Check the block excuted code for details:

{:lang='ruby'}
	if options[:yield]
	  block.call(base)
	else
	  base.instance_eval(&block)
	end

Internal of `on_load`, it receive three params `name, options={}, &block`. we can pass a hash options for it, and then excute block will check if it contains `yield` option. Here `base` is __ActiveRecord::Base__, according `yield` option to call the block or instance_eval it.So, by passing ActiveRecord we can decide who will excute the block.

So we defined the block first, and then wait for the dependence to be loaded, so that we can excute the code later. People call it [Lazy Loading](http://en.wikipedia.org/wiki/Lazy_loading),

> which is a design pattern commonly used in computer programming to defer initialization of an object until the point at which it is needed.  

By using this feature, rails can load only key components first, faster the booting process.Defines the block 
