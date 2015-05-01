---
layout:     post
title:      "Tealeaf Academy Week4 Reflection"
subtitle:   ""
date:       2015-02-23 13:42:00
author:     "Overlord"
header-img: "img/tealeaf2.png"
tags:       ["Ruby on Rails", "Javascript", "Research", "Random Thoughts"]
---

##**Overview**
This is the last week of Tealeaf course two. Most of the materials this week is about taking thePostit project we built in week 3 and add some finishing touch to it. The topics include some intermediate level techniques such as slugs, Admin role, and time zone. It took me an extra two weeks to finish due to a nasty flu that kept me in bed for a week, plus a trip to Hong Kong that kept me away from my computer for a few days.

My final Postit app is up and running at: 

https://postit-eugene-chang.herokuapp.com/

The last assignment of the course is about creating my own rails application in 3 to 4 days, the process was both fun and struggling, luckily, I completed all the functions I want in time. 

My final project is up and running at:

<a href="https://duck-on-rails.herokuapp.com">https://duck-on-rails.herokuapp.com/ </a>

I'll make a blogpost about the my struggle during implemenation and the technical highlights of this app later.

##**Ajax and Rails**
A quick review about Ajax: Ajax is short for Asynchronous JavaScript + XML. From the programmer's point of view, Ajax allows programmers update part of the webpage's content/data without having to refresh the entire page. Which is great for enhancing user experience and reducing computational cost.
This week's materal is about rails-flavored Ajax, which is about making Ajax request in by using rails helper instead of jquery and other methods.

First, we add ```remote: true``` to the linker helper method :

{% highlight html linenos%}
<%= link_to vote_post_comment_path(comment.post, comment, vote: true), method: 'post', remote: true do %>
{% endhighlight %}

The ```remote: true``` will add ```data-remote="true"``` into the html tag. Now when I click on the link generated, rails will send the following request:

```shell
Started POST "/posts/9th-post/vote?vote=true" for 127.0.0.1 at 2015-03-05 11:17:59 +0800
Processing by PostsController#vote as JS
```
Notice the ```Processing by PostsController#vote as JS```, which means that this is a **javascript request**, and we need to handle this javascript request in the respective action of our controller. To handle different formats of response, we can add:

{% highlight ruby linenos%}
respond_to do |format|
  format.html do
    if @vote.valid?
      flash[:notice] = 'Your vote was counted.'
    else
      flash[:error] = 'You can only vote on a post once.'
    end
    redirect_to :back
  end
  format.js
end
{% endhighlight %}

Notice that if we leave format.js blank as it is shown in the above code, by default rails will render a template with the same name as the action, so in our case it will render a template callled ```vote``` in our post controller, but it's not an html template, it's a javascript template, so we need to create a new file named: ```vote.js.erb``` under **views/posts/**

erb vote.js.erb
{% highlight ruby linenos%}
<% if @vote.valid? %>
  $("#post_<%= @post.slug %>_votes").html("<%= @post.total_votes %> votes")
<% else %>
  alert('You can only vote on a post once.');
<% end %>
{% endhighlight %}

The javascript code shown above is pretty straightforward, it's about replacing what's inside the html tag that has ```id``` equals to ``` post_<%= @post.slug %>_votes``` with html code:
```erb
<%= @post.total_votes %> votes
```

##**Slugs**
Slug, or semantic url, is a technique to transform url from pure numbers into more descriptive, readable words. Which not only make an web application more user friendly, it also helps improve the application's SEO (search engine optimization) and protects the url from leaking informations related to the application or database. For example, ```postit/users/8``` will give away the that there are aorund 8 users registered for this particular web application.

So we want to change ```postit/users/8``` into  ```postit/users/eugene-chang``` in our url when we're displaying the user profile. We can start by

##**Extract Common Logic to Module**
Sometimes we have repeated logic and code across different models, for example, our slug function. Having repeated code across different models doesn't just made the code more complex, it also become more difficult to maintain later on. So it's best that we extract the common logic/code into a module, and just include the module for the model class that we need to use it, and it's implementation is quite simple in rails:

We want to put our extracted common logic under the folder ```app/lib```, so we need to configure the path for rails to look for in ```application.rb``` file:

ruby /config/application.rb
{% highlight ruby linenos%}
config.autoload_paths += %W(#{config.root}/lib) #**find the file underneath root/lib
{% endhighlight %}

Now we can move all the logic related to slug into the module file under lo

ruby /lib/sluggable.rb

{% highlight ruby linenos%}
module Sluggable
  extend ActiveSupport::Concern

  included do #the code will be executed when this module is included
    before_save :generate_slug!
    class_attribute :slug_column
  end

  def to_param
    self.slug
  end

  def generate_slug!
    #self.slug = self.title.gsub(" ", "-").downcase
    the_slug = to_slug(self.send(self.class.slug_column.to_sym)) #change slug_column into symbol and pass it to to_slug method
    obj = self.class.find_by slug: the_slug
    count = 2
    while obj && obj != self #while post object exists, and it's not the same object we're dealing with, need to append a number to it
      the_slug = append_suffix(the_slug, count) #two posts with same title will create same slug, not good!
      obj = self.class.find_by slug: the_slug
      count += 1
    end

    self.slug = the_slug.downcase

  end

  def append_suffix(str, count)
    if str.split('-').last.to_i != 0
      return str.split('-').slice(0...-1).join('-') + "-" + count.to_s
    else
      return str + "-" + count.to_s
    end
  end

  def to_slug(name)
    str = name.strip
    str.gsub! /\s*[^A-Za-z0-9]\s*/, '-'
    str.gsub! /-+/, "-"
    str.downcase
  end

  module ClassMethods
    def sluggable_column(col_name)
      self.slug_column = col_name
    end
  end

end
{% endhighlight %}
Once I finished adding the module file, I can simply use the module by including it in the model:  

{% highlight ruby linenos%}
class Post < ActiveRecord::Base

  include Sluggable
  
end
{% endhighlight %}

##**Admin Role**
You definitely do not want all the users to be able to perform destructable actions. So it is better to set up a administration class users, and only grant these users the ability to perform destructable actions. This can be implemented by adding an 'admin' column to Users table, and add the following method to application controller:

{% highlight ruby linenos%}
 def require_admin
   access_denied unless logged_in? and current_user.admin?
 end
{% endhighlight %}

##**Conclusion**
After much delay (about 15 days) I am finally done with the material of this week. The week 4 material is focused on some techniques to make a rails app more sophisticated, such as slug, admin role, and time zone. The final project took me a few days to work on it. Although I ran into a few hurdles, the overall process is very fun. There is no better way to feel accomplished as a programmer other than being able to bring your concept and idea to life with the skills you master.