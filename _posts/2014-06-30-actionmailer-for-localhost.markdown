---
layout: post
title:  "Configuring Action Mailer on your Development environment"
date:   2014-06-30 08:00:00
categories: rails actionmailer
tags: [rails, actionmailer, mandrill]
---

This article will help you get up and running with Action Mailer on your local machine.  A follow up post will cover deploying to Heroku.

Action Mailer by default sends a multipart email (HTML and plain text) which is perfect.  You should always send this way unless you have a specific use case.  I prefer to use a third-party provider for the sending step and I recommend the same.  For this project I wanted to test out [Mandrill][mandrill] but feel free to use whatever provider you want.  SendGrid and Mailgun are excellent, both offer a free tier but if you want to look around, Heroku has a good list of providers [here][heroku].

[mandrill]: http://mandrill.com/
[heroku]: https://addons.heroku.com/#email-sms

> You can use your Gmail account but I’m going to caution against it.  You should protect your Gmail with [2FA][google-2fa] at a minimum and [App specific passwords][google-apppass] for something like this.  Still, don’t do it unless you have a good reason.

[google-2fa]: https://support.google.com/accounts/answer/180744?hl=en
[google-apppass]: https://support.google.com/accounts/answer/185833?hl=en

I recommend starting a new rails project to see how this works before adding a mailer to an ongoing project.  Let’s get started, jump into your CLI and type:

{% highlight bash %}
$ rails new actionamiler -T
{% endhighlight %}

>-T doesn’t add the test framework which we won’t need, this is optional.

Next, `cd` into your new folder, `git init`, `git add --all` and `git commit -m “Initial commit”` (practice makes perfect with git).

Let’s start with a quick User model like so:

{% highlight bash %}
$ rails g model User name:string email:string -T
{% endhighlight %}

First things first, check your migration!  Make sure you didn’t fat finger any field names or miss something.  Next step is to run our migration:

{% highlight bash %}
$ rails db:migrate
{% endhighlight %}

Git time.  Let’s go ahead and create a quick controller so we can create a signup page.  Welcome and create are the only pages needed for this test.  I’m using the --no-assets and --no-helper option here because I don’t need the extra files.

{% highlight bash %}
$ rails g controller Users welcome create --no-assets —-no-helper
{% endhighlight %}

Make a quick change to config/routes.rb like this:

{% highlight ruby %}
Rails.application.routes.draw do
  resources :users, :only => [:create]
  root to: "users#welcome"
end
{% endhighlight %}

I prefer to _route_ the root to our users#welcome page and users#create will be our POST location.  There will be no validation on the form and no edit, destroy or show actions for simplicity.  My user controller has been configured to #create on the POST action and save to the database, simple.  Success or fail the controller just redirects with a simple flash notice on root (users#welcome).  Go ahead and test out everything to make sure it’s working as intended.  All good?  Time to git.

Next we generate our mailer:

{% highlight bash %}
$ rails g mailer welcome_mailer
{% endhighlight %}

This will create a file in app/mailers/ and a new folder in your views.  Let’s take a quick look at the new file welcome_mailer.rb.  Nothing much going on here but feel free to change the default from: to whatever from email you want.

{% highlight ruby %}
class WelcomeMailer < ActionMailer::Base
  default from: "hello@mysite.com"
end
{% endhighlight %}

We’re going to need a method to call to get the email ready so let’s add that.  I called mine `welcome_email` and I take one param, user (which is my new user object from #create).  I also set a few Class level instance variables to pass down some additional information to my view (aka template in the case of our mail).  `@user` will contain the new user record object and `@url` is a url back to your site.

{% highlight ruby %}
class WelcomeMailer < ActionMailer::Base
  default from: "hello@mysite.com"

  def welcome_email(user)
    @user = user
    @url  = 'http://test.com/login'
    mail(to: @user.email, subject: 'My awesome email')
  end
end
{% endhighlight %}

The last line of the method above will return a `Mail::Message` object which can then be triggered to send with the `deliver` method.

Let’s create 2 new files for the HTML and text version of the email.  These files go in app/views/welcome_mailer and I’ve called them welcome_email.html.erb and welcome_email.text.erb.  They should match your mailer class so name things appropriately here.

If you’re still following, take a quick break to `git`, coffee and stretch.  Next steps are the configure the mailer smtp settings, add in your mail password to the secrets.yml file then test.

Navigate to your config/environments/development.rb file and add the following before the end:

{% highlight ruby %}
config.action_mailer.perform_deliveries = true
config.action_mailer.raise_delivery_errors = true
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  :address              => "smtp.mandrillapp.com",
  :port                 => 587,
  :enable_starttls_auto => true,
  :user_name            => ENV['super_secret_username'],
  :password             => ENV['super_secret_password'],
  :domain               => ENV['super_secret_domain'],
  :authentication       => 'plain'
}
{% endhighlight %}

Your information above may be slightly different depending on the email provider you’re using.  DO NOT put your username or password for your email provider here, instead use `ENV[‘super_secret_username’]`, `ENV[‘super_secret_password’]`.  Since we’re here we might as well remove the domain to a secret as well `ENV[‘super_secret_domain’]`.  I’ll explain this in one minute.  Git now.

This next step is really important.  Open your .gitignore file and add this line:

{% highlight bash %}
# Ignore my secret stuff
/config/secrets.yml
{% endhighlight %}

Then git add and commit.  Navigate to your config/secrets.yml file and add the following under development:

{% highlight bash %}
super_secret_username: ‘your mail provider username’
super_secret_password: ‘your mail provider password’
super_secret_domain: ‘your send from domain’
{% endhighlight %}

Run git status, notice that your secrets.yml file isn’t part of the tracked changes anymore, this is crucial if you push to Github.  You definitely do not want to git push the secrets.yml file with sensitive info.  Epic fail, be careful.

![faceplam]({{ site.baseurl }}images/facepalm.jpg)

This next step tripped me up a little and made me pause for a rubber ducky moment.  When your rails environment loads you need to make sure to pull in the secrets.yml file.  This doesn't happen by default but a quick search led me to an [article][quickleft] explaining this.

[quickleft]:http://quickleft.com/blog/simple-rails-app-configuration-settings

Add this line to your config/application.rb file:

{% highlight ruby %}
ENV.update YAML.load_file('config/secrets.yml')[Rails.env] rescue {}
{% endhighlight %}

You’ll need to reload your rails environment to see the new changes so kill your server and run it again.  Let’s jump into the rails console to see if the secrets.yml loaded up for us.  Inside your console type `ENV` and you should get a big long hash.  Somewhere in there is going to be your secret variables, if not, something went wrong.  Retrace your steps and make sure you can see that in your ENV hash.

If everything looks good, git commit.

Run your server and test it out.  If everything’s good you should be sending email now.  If you don’t get emails or something isn’t working check your log and/or pry into whatever part of the chain broke for you.  Pry is a great tool and should give you enough info to figure out your next steps.






