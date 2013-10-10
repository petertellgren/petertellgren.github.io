---
layout: post
title: "Adding Dynamic View Paths to ActionMailer in Rails 2.3.11"
date: 2011-10-13 10:05
comments: true
categories: [Rails, ActionMailer, Views, Multi-Tenancy]
---

When I recently rewrote our app at work into a multi tenant app I ran in to the problem of the action_mailer views. Since the app started out it’s life as a single tenant app most of the mails had to be changed but also had to remain the same for the app still there

My first idea was to just configure the template_root of ActionMailer in my new Environment but just after this was done the requirement came that each tenant might want to customize these views. so instead of changing the template_root depending on the tenant I googled around and found a solution for adding the nice view_path method we already have for the ActionController.

The original post can be found [here](http://www.quirkey.com/blog/2008/08/28/actionmailer-hacking-multiple-template-paths/) but I thought I would show my implementation below. Basically a compilation of the original poster and the comments given to that post

```ruby
module ActionMailer
  class Base
    class_inheritable_accessor :view_paths

    def self.prepend_view_path(path)
      view_paths.unshift(*path)
    end

    def self.append_view_path(path)
      view_paths.push(*path)
    end

    def view_paths
      self.class.view_paths
    end

    private

    def self.view_paths
      @@view_paths ||= ActionController::Base.view_paths
    end

    def initialize_template_class_without_helpers(assigns)
      ActionView::Base.new(view_paths, assigns, self)
    end

  end
end
```

I made sure that my lib directory was included in my load paths and then I could mange the view paths as follows

```ruby
ActionMailer::Base.prepend_view_path(RAILS_ROOT + '/app/views/...')
``` 