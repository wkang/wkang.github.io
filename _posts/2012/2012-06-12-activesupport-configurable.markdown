---
title: ActiveSupport Configurable 
date: 2012-06-12
slug: 2012/06/12/activesupport-configurable
layout: post
categories: notes
tags: [ruby activesupport]
desc:  ActiveSupport::Configurable store options with OrderHash
---
ActiveSupport::Configurable let us store options with [ActiveSupport::OrderHash](http://api.rubyonrails.org/classes/ActiveSupport/OrderedHash.html)(Ruby1.8 is undefined, 1.9 did) in a class or object.
Let' see how we can do that:

{:lang='ruby'}
	require "active_support/configurable"

	class Witch
		include ActiveSupport::Configurable

		config_accessor :weapon, :instance_reader => true, :instance_writer => false
		config_accessor :helmet, :instance_reader => false, :instance_writer => true

        # initialize config block
		configure do |c|
			c.weapon = 'Fire Truncheon'
			c.helmet = 'Iron Helmet +1'
		end
	end

With'config'hash, we can edit options at will in class or instance.  
`config_accessor`allows you to add shortcut so that you don’t have to refer to attribute through config.  
`instance_reader`and`instance_writer`make objects can read and write the attribute.From above code, `Witch`class and objects have`weapon`and`helmet`attributes:  

{:lang='ruby'}
    Witch.config 
    #{:weapon=>"Fire Truncheon", :helmet=>"Iron Helmet +1", :ring=>"Ice ring"}
    Witch.config.weapon = "Truncheon" 
    #{:weapon=>"Truncheon", :helmet=>"Iron Helmet +1", :ring=>"Ice ring"}
    Witch.new.config.weapon = "Truncheon +10"

    Witch.weapon #Fire Truncheon  
    Witch.helmet #Iron Helmet +1  

    john = Witch.new
    john.weapon #Fire Truncheon

    john.weapon = "Royal Truncheon +1"
    #activesupport_configurable.rb:26:in `<main>': 
    #undefined method `weapon=' for #<Witch:0x94136ac @_config={}> (NoMethodError)

    john.helmet = "Normal Helmet"
    john.helmet #Normal Helmet

    #helmet have no reader method
    john.helmet  
    #activesupport_configurable.rb:30:in `<main>':  
    #undefined method `helmet' for #<Witch:0x9999cac @_config={:weapon=>"Royal Truncheon +1"}> (NoMethodError)

## Inside the source code

{:lang='ruby'}
    reader, line = "def #{name}; config.#{name}; end", __LINE__
    writer, line = "def #{name}=(value); config.#{name} = value; end", __LINE__

    class_eval reader, __FILE__, line unless options[:instance_reader] == false
    class_eval writer, __FILE__, line unless options[:instance_writer] == false

we can see that it adds reader and writer instance methods to the class by `class_eval`,

## Shortage
*    ### can not persist config options, try [rails3_settings](https://github.com/midwire/rails3_settings)
*    ### can not support YAML file, try [settingslogic](https://github.com/binarylogic/settingslogic) 
