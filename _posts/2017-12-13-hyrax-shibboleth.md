---
layout: post
title: Shibboleth Authentication for Hyrax
author: David Chandek-Stark
---

In a [previous article](/2015/02/11/authentication-with-devise-omniauth-and-shibboleth) I discussed a strategy for integrating
Shibboleth authentication with a (then-called) Hydra application (that Hydra project has since been renamed to [Samvera](https://samvera.org/)).
The implementation details of this strategy in the [Duke Digital Repository](https://repository.duke.edu) have evolved slightly in the interim,
hopefully in the direction of improvement and simplification, and, now that we are engaged in a pilot project built on [Hyrax](http://hyr.ax/) 2.0, it
seems useful to update this information.

**Important!** Since we intend to change the Hyrax `user_key` from its default field `email`, implementers are advised to start with clean user data if possible. We have observed issues in transitioning users from registrations based on email to Shibboleth identity information, at least where the email address differed from the person's [EPPN](https://www.internet2.edu/media/medialibrary/2013/09/04/internet2-mace-dir-eduperson-201203.html#eduPersonPrincipalName). This article does not cover how to work through those issues.

### The Gemfile

Add `gem 'omniauth-shibboleth'` to our Gemfile and `bundle install`.

### The Database Migration

We need to add a field to store our `uid` value in the `users` table.
The name of the field is not critical; choose your own and substitute accordingly in the rest of this article.
You may or may not want or need to index the field, depending on the number of users you expect to support.
If you set `null: false` be sure to also provide a default value, say `default: ""`.
We will use Rails validations in our example model to ensure presence and uniqueness.


{% highlight ruby %}
class AddUidToUsers < ActiveRecord::Migration[5.1]
  def change
    change_table :users do |t|
      t.string :uid, index: true
    end
  end
end
{% endhighlight %}

### The Devise Initializer

Now we update the Devise configuration set in its initializer.

{% highlight ruby %}
# config/initializers/devise.rb

# ==> Configuration for any authentication mechanism
# Configure which keys are used when authenticating a user.
# ...
config.authentication_keys = [:uid]

# You may also want to update these settings ...
config.case_insensitive_keys = [:uid]
config.strip_whitespace_keys = [:uid]

# ==> OmniAuth
# Add a new OmniAuth provider. Check the wiki for more information on setting
# up on your models and hooks.
# ...
require 'omniauth-shibboleth'
config.omniauth :shibboleth, {
		  uid_field: lambda { |rpm| rpm.call("eppn") || rpm.call("duDukeID") },
		  name_field: "displayName",
		  info_fields: {
		    email: "mail",
		  },
		  extra_fields: ["duDukeID"],
		}
{% endhighlight %}

The `uid_field` mapping lambda indicates that we will use `eppn` if available, or the field `duDukeID` (a custom institutional identifier) otherwise.
See [omniauth-shibboleth documentation](https://github.com/toyokazu/omniauth-shibboleth#more-flexible-attribute-configuration)
for details on this type of configuration.
If you just want to use `eppn` as the uid, then write `uid_field: "eppn"`, which is also the omniauth-shibboleth default.
The `name_field` mapping to `displayName` is the default for omniauth-shibboleth, but we'll be explicit here.

### The Controller

Now we need to implement the `shibboleth` action in `OmniauthCallbacksController`.  This is the bit that uses the authentication information from Shibboleth
to sign in with Devise.

We can generate the controller in the usual way with the Devise generator `rails generate devise:controllers users`. At the moment we only care about `OmniauthCallbacksController`; you may discard the other generated controllers as you see fit.

After adding the new action, here's our updated controller:

{% highlight ruby %}
# app/controllers/users/omniauth_callbacks_controller.rb

class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController

  def shibboleth
    # We have to implement this class method in our User model, below.
    user = resource_class.from_omniauth(request.env["omniauth.auth"])
    set_flash_message :notice, :success, kind: "Duke"
    sign_in_and_redirect user
  end

end
{% endhighlight %}

In order to route requests properly to our new controller, we need to update our routes config:

{% highlight ruby %}
# config/routes.rb

devise_for :users, controllers: { omniauth_callbacks: "users/omniauth_callbacks" }
{% endhighlight %}

### The Model

In our `User` model we first need to activate the Devise omniauth module and specify the omniauth provider:

{% highlight ruby %}
# app/models/user.rb

devise :database_authenticatable, :registerable, :recoverable,
       :rememberable, :trackable, :validatable,
       :omniauthable, omniauth_providers: [:shibboleth]							       
{% endhighlight %}

Note that we are leaving `:database_authenticatable` active in order to support development and testing requiring user authentication outside of a Shibboleth
integration environment (e.g., workstation, CI, etc.).  The other modules shown may be part of the default Devise installation but are not significant here.

Next we need to implement a class method in our `User` model that returns a User instance to sign in.
There's nothing magic about the method name `from_omniauth`; choose a different name if you like. 
We will assume that "registration" of new users via Shibboleth authentication is automatic -- i.e.,
that we will create a new `User` if necessary for the provided credentials.

{% highlight ruby %}
# app/models/user.rb

# @param auth [OmniAuth::AuthHash] authenticated user information.
# @return [User] the authenticated user, possibly a newly created record.
# @see {Users::OmniauthCallbacksController#shibboleth}
def self.from_omniauth(auth)
  find_or_initialize_by(uid: auth.uid).tap do |user|
    # set a random (unusable) password for a new user
    user.password = Devise.friendly_token if user.new_record?
    
    # set/update attributes based on our omniauth-shibboleth authentication mapping
    # see https://github.com/omniauth/omniauth/wiki/Auth-Hash-Schema
    user.update!(email: auth.info.email, display_name: auth.info.name)
  end
end			    
{% endhighlight %}

In order to integrate smoothly with Hyrax and its dependencies (e.g., Blacklight), we will change the `#to_s` and `#user_key` methods:

{% highlight ruby %}
# app/models/user.rb

# Method added by Blacklight; Blacklight uses #to_s on your
# user class to get a user-displayable login/identifier for
# the account.
def to_s
  # We prefer the user's actual name ("David Chandek-Stark") to their uid,
  # if available. This is of course up to you.
  display_name || uid
end

def user_key
  uid
end
{% endhighlight %}

Finally, you may wish to implement validation of the `uid` attribute, although it's not required.

{% highlight ruby %}
# app/models/user.rb

validates :uid, presence: true, uniqueness: true
{% endhighlight %}

### Views

For the Shibboleth integration piece itself, we don't need to customize any of the Devise views because users on our site will not use registration or login forms provided by our application. However, if we retain `database_authenticatable` functionality for development and testing, then we need to update the default views to work with the changes we have made.  Follow the Devise documentation for generating views to customize.  You will have to replace references to `email` with `uid`, or add `uid` as a required field to `devise/sessions/new.html.erb`, etc.  Note that if you generate views in the `users` scope, you will also need to update the routing configuration.

### The Web Server

In our example we assume a Rails production environment under Passenger + Apache (2.4) with [mod_shib](https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPApacheConfig). Hopefully readers can translate well enough to other configurations for this discussion to be useful.

The relevant bits of the virtual host configuration (not intended as a complete Rails/Passenger config example):

```
# The Location block and RewriteRule just prevent problems :)
RewriteEngine on
<Location /Shibboleth.sso>
  PassengerEnabled off
</Location>
RewriteRule ^/Shibboleth.sso - [L]

# Rails
DocumentRoot /path/to/rails/root/public
<Directory /path/to/rails/root/public>
  AuthType Shibboleth
  AuthName "NetID"
  
  # This configuration specifies that mod_shib is enabled
  # but the user is not required to login.
  # If you want to force login, set requireSession to 1
  # and use Require directive.
  # See https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPApacheConfig.
  ShibRequestSetting requireSession 0 
  Require shibboleth
</Directory>

# OPTIONAL: Automatic routing of sign in through Shibboleth
# If you omit this directive, users have to click "Login with Shibboleth"
# on the login form.
Redirect /users/sign_in /users/auth/shibboleth

# REQUIRED, unless you require login to access Rails (above)
<Location /users/auth/shibboleth>
  ShibRequestSetting requireSession 1
</Location>

# OPTIONAL: Auto-logout of Shibboleth SSO.
# You may prefer using `after_sign_out_path_for(scope)` in
# a customized Devise SessionsController instead.
<Location /users/sign_out>
  # You may want or need to customize the value of the Location header set here
  Header edit Location ^.* /Shibboleth.sso/Logout "expr=%{REQUEST_STATUS} == 302"
</Location>
```

### nginx Notes

nginx + shib + omniauth config may require setting the omniauth-shibboleth option `request_type` to `:header` in one or more configuration files:

Devise

{% highlight ruby %}
# config/initializers/devise.rb

config.omniauth :shibboleth, request_type: :header, ... 
{% endhighlight %}

Rack Middleware

{% highlight ruby %}
# config/initializers/omniauth.rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :shibboleth, request_type: :header
end
{% endhighlight %}

(Credit: eefahy)

### The End

Enjoy!