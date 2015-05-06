---
layout:     post
title:      "Tealeaf Academy Course2 Week3 Reflection"
subtitle:   ""
date:       2015-02-02 13:42:00
author:     "Overlord"
header-img: "img/tealeaf2.png"
tags:       ["Ruby on Rails", "Research"]
comments:   true
---

##**Overview**
The most important topic covered this week is the concept and implementation of Authenication using Ruby on Rails. Authenication is one of the major functions that separates a professional application from an amatuer/toy application. From my past experience of programming ASP.NET, my limited understaning of authenication is a simple string comparison by doing:

{% highlight ruby linenos%}
user_pass = sql_execute( select user.password from user_table where user.name = "input_username" )
if user_pass == input_userpassword 
  Authorized
else
  Warning
end
{% endhighlight %}

After going through all the material this week, I realized how naive and limited my comprehension is on the subject.
Other important topics include the polymorphic association, where in a 1:M association, a model can belongs_to multiple models with a single association, this is critically important because on the databse level, it not only keeps the data schema simple and clean, it also greatly reduces the wasted space. 
Another important topic is the **Member / Collection** keyword, which allows user to add more routes than the default seven RESTful routes set by rails. Concepts of Memoization and flash are also introduced.

##**Authenication**
Chris mentioned an important concept about authenication during lecture, and that is **one-way hashing**. For modern authenication systems, storing a user's actual password in database can lead to major security issues. So it is better to use one-way hashing method, which is to use hashing algorithms to turn a user input password into a unique string, and compare the hashed strings instead of the actual password everytime when users try to log in. In Rails, there is a built-in function in ActiveModel called **has_secure_password**, which will automatically set and authenicate Bycryt password. **has_secure_password** requires an attribute caled **password_digest** in the same model in order to work, which stores the unique string that is hashed by Bycrpt algorithm. 

The implementation include the following steps:

1. add **'bcrypt-ruby'** gem to Gemfile:

{% highlight ruby linenos%}
gem 'bcrypt-ruby', '~> 3.0.0'
{% endhighlight %}

Then run ``bundle install``.

2. generate a migration file and add **password_digest** column to the **users** table in database.

{% highlight ruby linenos%}
def change
  add_column :users, :password_digest, :string
end 
{% endhighlight %}

Then run ``rake db:migrate``.

3. **has_secure_password** adds methods to set and authenticate against a BCrypt password, it has built-in validation frameworks, but for our project we want to build our own, so we set the built-in validation to false and add our own validation, which are username and password. Which can be written as the following:

{% highlight ruby linenos%}

class User < ActiveRecord::Base
 
 has_secure_password validations: false
 
 validates :username, presence: true, uniqueness: true
 validates :password, presence: true, on: :create, length: {minimum: 5}

end
{% endhighlight %}

*Notice that setting **has_secure_password** will give us two virtual attributes for the user model, which are **password** and **password_confirmation**, they do not appear in the database layer nor the model, but they are essential to the mechanism of **has_secure_password**.

##**Validation**
After we set up the the autehnication in the user model, we can add some mehtods in applicaiton controller to help perform validation in our application:

```application_controller.rb```

{% highlight ruby linenos%}
class ApplicationController < ActionController::Base
  # Prevent CSRF attacks by raising an exception.
  # For APIs, you may want to use :null_session instead.

   protect_from_forgery with: :exception

   helper_method :current_user, :logged_in?
   #make these methods available to view templates

   def current_user
     @current_user ||= User.find(session[:user_id]) if session[:user_id]
     #memorization technique, if @current_user object is nil, and session hash has an user_id, then find the user in the database with the same user_id from session hash.
   end

   def logged_in?
     !!current_user #check if there is a user, !! turns any object into a boolean value.
   end

   def require_user #this method is used to restrict actions that require authenication
     if !logged_in?
       flash[:notice] = "You must be logged in to do that!"
       redirect_to :back
     end
   end
end
{% endhighlight %}

Notice that **helper_method** will make these methods accessible to views.
  validates_uniqueness_of :scope

##**Routing: Members and Collection**
Sometimes we need to create actions that are outside the default seven RESTFUL actions provided by Rails, this is where **Members** and **Collection** keywords come in. In short, **members** is used to add addtional action to a specific member of the resource, and collection is used add additional action to a collection of resource. 

|URL                |URL helper                 |Action         |
| ----------------- |:-------------------------:|--------------:|
|/photos/1/preview  | preview_photo_path(photo) | photos#preview|  
|/posts/archive     | archive_photos_path       | photos#archive| 

In the case of our postit project:

{% highlight ruby linenos%}
resources :posts, except: [:destroy] do
  member do  
    post :vote  
  end
{% endhighlight %}

will give us the following routes: 

|path             |url helper              |action         |http verb|
| --------------- |:----------------------:| :------------:|--------:|
|/posts           | posts_path             | posts#index   |  GET
|/posts/new       | new_posts_path         | posts#new     |  GET
|/posts/:id       | post_path(@post)       | posts#show    |  GET
|/posts/:id/edit  | edit_post_path(@post)  | posts#edit    |  GET
|/posts/:id       |                        | posts#create  |  POST
|/posts/:id       |                        | posts#update  |  PATCH
|/posts/:id       |                        | posts#update  |  PUT
|/posts/:id/vote  | vote_post_path(@post)  | posts#vote    |  POST


**Polymorphic Association**
One of the most important concepts of this week is **Polymorphic Association**. With polymorphic associations, a model can belong to more than one other model, on a single association. Traditionally, if we want to associate a model and multiple other models with 1:M associations, then
on the database level, the table will look like:

|id   |user_id  |vote    |post_id |comment_id |
| --- |:-------:|:------:|:------:|----------:|
|0    |1        |true    |1       |           
|1    |1        |false   |        |2        
|2    |1        |true    |2       |           

However, this will waste lots of space on the database level, so an alternative and better way of doing this would be:


|id   |user_id  |vote    |voteable_id |voteable_type |
| --- |:-------:|:------:|:----------:|-------------:|
|0    |1        |true    |1           |post          |
|1    |1        |false   |1           |comment       |
|2    |1        |true    |2           |post          |

Notice the **able** must be added to the column name for the polymorphic association to work.
The associations in the models should be set up like the following:

```vote.rb```
{% highlight ruby linenos%}
class Vote < ActiveRecord::Base
  belongs_to :voteable, polymorphic: true
end
{% endhighlight %}

```post.rb```
{% highlight ruby linenos%}
class Post < ActiveRecord::Base
  has_many :votes, as: :voteable
end
{% endhighlight %}

```comment.rb```
{% highlight ruby linenos%}
class Comment < ActiveRecord::Base
  has_many :votes, as: :voteable
end  
{% endhighlight %}

##**Memoization**
Since this is an important topic I'm gonna make a separate blog post about it.

##**flash[:notice] vs flash.now**
The flash is a special part of the session which is cleared with each request. This means that values stored there will only be available in the next request, which is useful for passing errormessages. It is accessed in much the same way as the session, as a hash ``flash[:notice]``. By default, adding values to the flash will make them available to the next request, but sometimes you may want to access those values in the same request.

Notice that there are two different flash methods, but there's a simple rule of thumb to determine how each method should be used: 

When redirecting use: 

{% highlight ruby linenos%}
flash[:notice] = "This message value is available in next request-response cycle"
{% endhighlight %}

When rendering use:

{% highlight ruby linenos%}
flash.now[:notice] = "Message is available in same request-response cycle"
{% endhighlight %}

##**Blank Comment Blows up**

##**Conclusion**

After going through this week's lesson, I had a much better understanding of the concept and implementation of authenication functions and realizing how little I used to know about it. 
The polymorphic association 