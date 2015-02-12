---
layout: post
title: Authentication with Devise, Omniauth and Shibboleth
author: David Chandek-Stark
tags: hydra ruby rails authentication shibboleth devise omniauth
---

In planning to roll out a public access Hydra head for our repository, we had these authentication and authorization requirements:

- Permit anonymous access
- When downloading a file requires authentication and the user is not logged in, display a "Login to Download" link. After authenticating the user should be redirected back to the original page.
- When a user who is not logged in attempts to access a restricted resource, force them to authenticate (i.e., rather than return a 403 unauthorized response).
- Require Shibboleth for authentication in production (on the server)
- Permit database authentication in development and test environments

## Background

In developing our first Hydra head, an administrative application for Library staff, the requirements were simpler:

- Require authentication
- Require Shibboleth for authentication in production (on the server)
- Permit database authentication in development and test environments

Since we would not permit anonymous access to the admin app, we could simply force Shibboleth authentication in the web server and then auto-login in Rails if the REMOTE_USER environment variable was set. To accomplish this result with Devise as the authentication framework, I developed a plugin, [devise-remote-user](https://github.com/duke-libraries/devise-remote-user). Using Devise's and Warden's failover capability, authentication in development and test would fallback to the database authenticatable strategy. Done.

Initially, to deal the public access case I attempted to modify devise-remote-user, but eventually realized that I would essentially have to reproduce what Devise + Omniauth provides. And fortunately, there is a fine [Shibboleth strategy for Omniauth](https://github.com/toyokazu/omniauth-shibboleth) already available. The complication I discovered is that Devise/Omniauth sort of assumes that Omniauth provider logins are presented as options on the standard user/password login form, unless omniauthable is the *only* authentication strategy you're using. In our case, I wanted the Shibboleth login to be automatic and seamless in production (on the server).

## Solution

### The Rails part

- We need a config setting to indicate whether we want to force Shibboleth authentication -- i.e., redirect requests from the :new action of the SessionsController to the Shibboleth provider method in the OmniauthCallbacksController. In our case this is a module class variable `require_shib_user_authn`, and it will be set to `true` in the production environment on the server.

- We need a config setting for the Shibboleth logout URL to use as the `after_sign_out_path_for(scope)` when Shibboleth authn is required. We decided to call this setting `sso_logout_url`.

- Override the Devise SessionsController to automatically route authentication to omniauth-shibboleth when required. This way we can use the standard `new_user_session_path` helper in views, keeping the branching logic in the controller.

{% highlight ruby %}
class Users::SessionsController < Devise::SessionsController

  def new
    store_location_for(:user, request.referrer) # return to previous page after authn
    if Ddr::Auth.require_shib_user_authn
      # don't want the "sign in or sign up" flash alert
      # which is set by the Devise failure app
      flash.discard(:alert)
      redirect_to user_omniauth_authorize_path(:shibboleth)
    else
      super
    end
  end

  def after_sign_out_path_for(scope)
    return Ddr::Auth.sso_logout_url if Ddr::Auth.require_shib_user_authn
    super
  end

end
{% endhighlight %}

- Override the Devise OmniauthCallbacksController in the standard way.

{% highlight ruby %}
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  
  def shibboleth
    # had to create the `from_omniauth(auth_hash)` class method on our User model
    user = resource_class.from_omniauth(request.env["omniauth.auth"])
    set_flash_message :notice, :success, kind: "Duke NetID"
    sign_in_and_redirect user
  end

end
{% endhighlight %}

- Create a custom failure app. This is probably not technically necessary due to the redirect in the SessionsController, but in the case of the admin app, where authn is required, we could just use this and not override SessionsController#new.

{% highlight ruby %}
class FailureApp < Devise::FailureApp

  def respond
    if scope == :user && Ddr::Auth.require_shib_user_authn
      store_location!
      redirect_to user_omniauth_authorize_path(:shibboleth)
    else
      super
    end
  end

end
{% endhighlight %}

Don't forget to configure the Warden failure app in the Devise initializer.

### The Apache part

    <Directory /path/to/rails/app/public>
      Options -MultiViews
      AuthName "NetID"
      AuthType Shibboleth
      # Permits anonymous access
      ShibRequestSetting requireSession 0
      Require shibboleth
    </Directory>

    # user_omniauth_authorize_path(:shibboleth)                                                                        
    <Location /users/auth/shibboleth>
      ShibRequestSetting requireSession 1
    </Location>
