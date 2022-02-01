---
# try also 'default' to start simple
theme: light-icons

# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
---

# Passwordless Rails Authorization

Simplify the user's experience

Sending the user a URL with a Token (MagicLinks)

Bill Tihen

---
layout: dynamic-image
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
left: true
---

# Demo

Demo Workflow wanted

* without login only landing page is available
* login only asks for email
* sends url via email with embedded token
* visiting URL with token creates a user session
* logout destroys session (only landing page is available)


---
layout: image-left
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---

# Token Choices

* **Rails Global-ID** - https://github.com/rails/globalid
* **JWT (JOSE)** - https://jwt.io/
* **PASETO** - https://paseto.io/
* **Branca** - https://branca.io/
* **Macaroons** - http://macaroons.io/
* **Biscuit** - https://github.com/biscuit-auth/biscuit
* ...

---
layout: image-right
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---
# Simple Global-ID Usage

```ruby
bin/rails c
# LOGIN ACCESS CODEs (from users email)
user = User.first # User.find(email: params[:email])
# create global id
sgid = user.to_sgid(expires_in: 5.minutes, for: 'user_access')
 # extract token from the Global ID
access_token = sgid.to_s
# build an access_url with the token
access_url = Rails.application.routes.url_helpers
                  .session_auth_url(token: global_id.to_s)
# Send the access_url in an email
UserAuthMailer.send_url(user, url).deliver_now

# VERIFY ACCESS
auth_token = params[:token]
auth_user = GlobalID::Locator.locate_signed(auth_token, for: 'user_access')
```

---
layout: image-left
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---

# Rails Configuration

```ruby
# config/environments/development.rb
self.default_url_options = { host: 'localhost', port: 3000 }
config.action_mailer.smtp_settings = { address: 'localhost', port: 1025 }

# congif/routes.rb
Rails.application.routes.draw do
  resources :users
  get  'login',      as: 'new_login',      to: 'sessions#new'
  post 'login',      as: 'login',          to: 'sessions#create'
  get 'auth/:token', as: 'session_auth',   to: 'sessions#auth'
  get 'logout',      as: 'session_delete', to: 'sessions#destroy'
  get 'landing',     as: 'landing',        to: 'landing#index'
  root to: "landing#index"
end
```

---
layout: image-left
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---

# Application Controller

Simple User Authorized Session Detection

```ruby
# app/controllers/users/application_controller.rb
class ApplicationController < ApplicationController
  before_action :users_only

  def current_user(user_id = session[:user_id])
    @current_user ||= User.find_by(id: user_id)
  end

  private
  # send unauthorized users to the root page
  def users_only
    return if current_user.present?

    redirect_back(fallback_location: root_path)
  end
end
```

---
layout: image-right
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---
# Session Authorization

Create a Session from Token (Magic Link) compatible with `current_user`

```ruby
# touch app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  skip_before_action :users_only, only: [:new, :create, :auth]
  def auth
    user_token = params[:token].to_s # to_s prevents nil error
    user = GlobalID::Locator.locate_signed(user_token, for: 'user_access')
    if user
      session[:user_id] = user.id # create the session as designed
      redirect_to user_path(user), notice: "Welcome back #{user.name}"
    else
      redirect_to(root_path, alert: "login link needed")
    end
  end
  def destroy # logout
    session.clear
    redirect_to(root_path, notice: "logout successful")
  end
end
```

---
layout: image-left
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---
# Token Prep / Senden

Create a Session from Token (Magic Link) compatible with `current_user`

```ruby
# touch app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  skip_before_action :users_only, only: [:new, :create, :auth]
  def new
    render :new, locals: {user: User.new}
  end
  def create
    user_params = params.require(:user).permit(:email)
    user = User.find_by(email: user_params[:email])
    if user
      global_id = user.to_sgid(expires_in: 15.minutes, for: 'user_access')
      access_url = session_auth_url(token: global_id.to_s)
      UserAuthMailer.send_url(user, access_url).deliver_later
    else # do the same logic here to make time the same
    end
    redirect_to(root_path, notice: "Access-Link has been sent")
  end
end
```

---
layout: image-right
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---

# Choosing a Token Tech

Lots to consider:

* Interoperability
* Availability in Ruby
* Features (Access Control, etc)
* Security (algorithm agility - is a weakness)

**Rails-Global-ID** <small>(perfect for stand-alone rails apps)</small>
* simple to use
* nothing to install
* no migrations needed

---
layout: image-left
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---

# Summary

* Auto-expiring
* No migrations needed
* Simple user experience
* As secure as the users email
* Users find this very easy to use
* Works with Devise and other User Management tools

---
layout: image-right
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---

# Security Considerations

* Throttle requests _(per IP)_
* Consider SGID expiration time
* Consider session expiration time
* Success and failure time during Auth should be equivalent _(to prevent E-Mail mining)_
* Adaptable as an extension on existing Auth Gems<br>_(secure & comes with pre-built account management!)_

---
layout: image-left
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---

# Implementations & Articles

<small>
<ul>
<li><b>Services</b> - OmniAuth, Keycloak, Google Auth, etc</li>
<li><b>via SMS</b> - https://dev.to/phawk/password-less-auth-in-rails-4ah</li>
<li><b>Passwordless Rails Gem</b> - https://github.com/mikker/passwordless</li>
<li><b>Extending Devise</b> - https://www.mintbit.com/blog/passwordless-authentication-in-ruby-on-rails-with-devise</li>
<li><b>Extending Sourcery</b> - https://fullstackheroes.com/rails/sorcery-passwordless-authentication/</li>
<li><b>Passwordless w/ an API</b> - https://blog.kiprosh.com/implement-passwordless-authentication-via-magic-link-in-rails-api/</li>
<li><b>Macaroons</b> - Improved Cookies for Authentication and Authorization</li>
</ul>
</small>

---
layout: image-right
image: ./images/david-herron-S5jD0E8DOC0-unsplash.jpg
---

# Questions? / Discussion

---

# Source - Code & Slides

* **Rails Code:**

  https://github.com/btihen/ruby_kafi_token_auth_rails_diy_code

* **Slides** (start with `yarn slidev`):

  https://github.com/btihen/ruby_kafi_token_auth_rails_diy_slides

---

# Appendix 0 - Session Expiration

By default rails sessions have no expiration (until logout).

To change this default behavior, you can set the session length with the setting:

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store, expire_after: 14.days
```


---

# Appendix 1 - Token Summary

<small>
<table>
  <thead>
    <tr>
      <th><b>Technology</b></th>
      <th><b>Features</b></th>
      <th><b>Ruby URL</b></th>
      <th><b>Interoperable Libraries</b></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><b>Rails Global-ID</b></td>
      <td>Built-into Rails</td>
      <td><small>https://github.com/rails/globalid</small></td>
      <td>None</td>
    </tr>
    <tr>
      <td><b>JWT (JOSE)</b><br><small>https://jwt.io/</small></td>
      <td>Well known & ubiquitous</td>
      <td><small>https://github.com/jwt/ruby-jwts</small></td>
      <td>~All:<br><small>https://jwt.io/libraries</small></td>
    </tr>
    <tr>
      <td><b>PASETO</b><br><small>https://paseto.io/</small></td>
      <td>less Algorithmic Agility than JOSE</td>
      <td><small>https://github.com/mguymon/paseto.rb</small></td>
      <td>Popular:<br><small>C, Elixir, Go, Rust, Ruby, Python, .NET, JS, Swift, Java</small></td>
    </tr>
    <tr>
      <td><b>Branca</b><br><small>https://branca.io/</small></td>
      <td>NO Algorithmic Agility</td>
      <td>NONE<br><small>https://github.com/tuupola/branca-spec</small></td>
      <td>Few:<br><small>Elixir, Python, PHP, JS</small></td>
    </tr>
    <tr>
      <td><b>Macaroons</b><br><small>http://macaroons.io/</small></td>
      <td>Allows layered restrictions & distributed Authorization</td>
      <td><small>https://github.com/localmed/ruby-macaroons</small></td>
      <td>Few:<br><small>C, C#, Go, Java, JS, Python, Rust, PHP</small></td>
    </tr>
    <tr>
      <td><b>Biscuit</b><br><small>https://github.com/biscuit-auth/biscuit</small></td>
      <td>Embeded RBAC</td>
      <td>NONE</td>
      <td>Few:<br><small>Rust, Web Assembly, Haskell, Java & Go</small></td>
    </tr>
  </tbody>
</table>
</small>

---

# Appendix 2 - Rails DIY Implementation

Basic Setup

```bash
rails new token_auth_rails_diy_code
cd token_auth_rails_diy_code
bin/rails db:create
bin/rails g controller Landing index
bin/rails g scaffold User name:string email:string
bin/rails db:migrate
```

Make a user at: http://localhost:3000/users

---
layout: image-right
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---
# Update Routes

Update the Routes (& add root_route):
```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :users
  get 'landing',     as: 'landing',        to: 'landing#index'
  root to: "landing#index"
ends
```

* Verify "/", "/landing" and "/users" are accessible.
* Make a user

---
layout: image-left
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---
# Restrict Access

To restrict access we will make a current_user and users_only in our ApplicationController.

```ruby
# app/controllers/users/application_controller.rb
class ApplicationController < ApplicationController
  before_action :users_only

  def current_user(user_id = session[:user_id])
    @current_user ||= User.find_by(id: user_id)
  end

  private
  # send unauthorized users to the root page
  def users_only
    return if current_user.present?

    redirect_back(fallback_location: root_path)
  end
end
```
<small>
Now verify both "/" and "/users" are inaccessible
</small>

```
This page isn’t working
slocalhost redirected you too many times.
```

---
layout: image-right
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---
# Landing Page Accessibility

To make the landing/root page accessible we use skip_before_actions:

```ruby
class LandingController < ApplicationController
  # `skip_before_action :users_only` allows access to the landing page
  skip_before_action :users_only, only: :index

  def index
  end
end
```

* Now we have fixed the routing loop
* "/" & "/landing" is accessible and
* "/users" is restricted (currently inaccessible)

---
layout: image-left
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---

# Session Create Routes

```ruby
# update routes: config/routes.rb
Rails.application.routes.draw do
  resources :users
  get 'auth/:token', as: 'session_auth',   to: 'sessions#auth'
  get 'logout',      as: 'session_delete', to: 'sessions#destroy'
  get 'landing',     as: 'landing',        to: 'landing#index'
  root to: 'landing#index'
ends
```

<small>`bin/rails routes | grep session`, should show:</small>
```bash
  session_auth  GET  /auth/:token(.:format)  sessions#auth
session_delete  GET  /logout(.:format)       sessions#destroy
```

---
layout: image-right
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---

# Session Create / Destroy

Now we can build a session (& grant access to `/users`).
```ruby
touch app/controllers/sessions_controller.rb

# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  skip_before_action :users_only, only: [:auth]

  def auth
    user_token = params[:token].to_s # to_s prevents nil error
    user = GlobalID::Locator.locate_signed(user_token, for: 'user_access')
    if user
      session[:user_id] = user.id # create the session as designed
      redirect_to user_path(user), notice: "Welcome back #{user.name}"
    else
      redirect_to(root_path, alert: "login link needed")
    end
  end
  # logout
  def destroy
    session.clear
    redirect_to(root_path, notice: "logout successful")
  end
end
```

---
layout: image-left
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---

# Test Token Auth and logout

```ruby
bin/rails c

# Configure URLS if not in environment settings
Rails.application.default_url_options = { host: 'localhost', port: 3000 }
# generate token
user = User.first
sgid = user.to_sgid(expires_in: 1.hour, for: 'user_access')
# embed token into a url (we will need make this routes)
url = Rails.application.routes.url_helpers
           .session_auth_url(token: sgid.to_s)
# copy url into browser (should have access to /users)

# logout (should no longer have access to /users)
http://localhost:3000/logout
```

---
layout: image-left
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---
# Login Mailer

Mail the auth_url to the user with the name of the server
```ruby
# generate the mailer
bin/rails generate mailer UserAuth send_url

# app/mailers/user_auth_mailer.rb
class UserAuthMailer < ApplicationMailer
  def send_url(user, auth_url)
    @user = user
    @auth_url = auth_url
    # # host_name - using dns name
    # @host_name = Rails.application.config.hosts.first
    # # host_name - using application name
    @host_name = Rails.application.class.module_parent_name
    mail(to: @user.email, subject: "Access-Link for #{@host_name}")
  end
end
```

---
layout: image-right
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---
# E-Mail Contents

Update the mail-templates with the send info.

```ruby
# HTML Template: app/views/user_auth_mailer/send_url.html.erb
<h1>Hi <%= @user.name %>,</h1>
<p><%= @host_name %> - <%= link_to "Access-Link", @auth_url %> </p>
<p> <%= @auth_url %> </p>
<p>This link is valid until: <%= DateTime.now + 15.minutes %>.</p>


# Text Template: app/views/user_auth_mailer/send_url.text.erb
Hi <%= @user.name %>,
Access-Link for <%= @host_name %> is:
<%= @auth_url %>
This link is valid until: <%= DateTime.now + 15.minutes %>..
```

---
layout: image-left
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---
# Test Email Code

```ruby
# configure: `config/environments/development.rb` with:
self.default_url_options = { host: 'localhost', port: 3000 }
# mail (mailhog) send config
config.action_mailer.perform_deliveries = true
config.action_mailer.smtp_settings = { address: 'localhost', port: 1025 }

bin/rails c
# include ActionView::Helpers::UrlHelper
# Rails.application.default_url_options = { host: 'localhost', port: 3000 }
user = User.first
sgid = user.to_sgid(expires_in: 15.minutes, for: 'user_access')
token = sgid.to_s
auth_url = Rails.application.routes.url_helpers
                .session_auth_url(token: token)
# use `deliver_now` in the console (in code use `deliver_later`)
UserAuthMailer.send_url(user, auth_url).deliver_now
# check mailhog / log & copy into browsers
```

---
layout: image-left
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---
# Setup Login Routes

Get a auth_url<br><small>`new` for the form, and `create` to generate the auth_url</small>
```ruby
# update the routes: config/routes.rb
Rails.application.routes.draw do
  resources :users
  get  'login',      as: 'new_session',    to: 'sessions#new'
  post 'login',      as: 'session',        to: 'sessions#create'
  get 'auth/:token', as: 'session_auth',   to: 'sessions#auth'
  get 'logout',      as: 'session_delete', to: 'sessions#destroy'
  get 'landing',     as: 'landing',        to: 'landing#index'
  root to: 'landing#index'
end
```

<small>`bin/rails routes | grep login`, should show:</small>
```bash
    new_login   GET  /login(:format)     sessions#new
        login   POST /login(:format)     sessions#create
```

---
layout: image-right
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---

# Login Form

All we need is the user's email so the form is quite simple:

```ruby
mkdir app/views/sessions
touch app/views/sessions/new.html.erb

# app/views/sessions/new.html.erb
<%= form_for(user, class: "user", # params with :user for secure_params
                   url: login_path, # give url when path and model differ
                   local: true, id: "login-form") do |form| %>
  <div class="field">
    <%= form.label :email %>
    <%= form.text_field :email %>
  </div>

  <div class="actions">
    <%= form.submit %>
  </div>
<% end %>
```

---
layout: image-left
image: ./images/balazs-busznyak-El5zuQAtfeo-unsplash.jpg
---

# Login in Session Controller

Now add the login (sending the token) code.
```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  skip_before_action :users_only, only: [:new, :create, :auth]
  def new
    render :new, locals: {user: User.new}
  end
  def create
    user_params = params.require(:user).permit(:email)
    user = User.find_by(email: user_params[:email])
    if user
      global_id = user.to_sgid(expires_in: 1.hour, for: 'user_access')
      access_url = session_auth_url(token: global_id.to_s)
      UserAuthMailer.send_url(user, access_url).deliver_later
    else
      # do the same logic here to make time the same (prevent user mining)
    end
    redirect_to(root_path, notice: "Access-Link has been sent")
  end
  ...
end
```

---

# Test full workflow

* go to: http://localhost:3000/login
* copy the login url into the browser
* logout

---

# Appendix 3 - Devise w/ DIY Session Auth

```ruby
rails new passwordless_devise_code
cd passwordless_devise_code

bin/rails db:create
bin/rails g controller Landing index
bin/rails g scaffold Pet name species

## config/routes.rb
Rails.application.routes.draw do
  resources :pets
  get 'home',    as: 'home',    to: 'pets#index'
  get 'landing', as: 'landing', to: 'landing#index'
  root to: "landing#index"
end
```

now the landing page and pets are freely available.

---

# Add Devise to add Authentication

```ruby
bundle add devise
bundle install
bin/rails generate devise:install
bin/rails generate devise User
# update the migration to match any added features
bin/rails db:migrate
```

---

# Basic Devise Config

```ruby
# config/environments/development.rb
self.default_url_options = { host: 'http://localhost:3000' }
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
# Rails.application.routes.default_url_options = { host: 'http://localhost:3000' }

# app/views/layouts/application.html.erb
<p class="notice"><%= notice %></p>
<p class="alert"><%= alert %></p>
```

---

# Basic Authentication Restrictions

Prevents all pages except 'landing', from non-authenticated access

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
end

# app/controllers/landing_controller.rb
class LandingController < ApplicationController
  skip_before_action :authenticate_user!
  def index
  end
end
```

---

# Configure Routes

If you want Token based logins and Password based logins (incase email fails) then you can simply leave Devise completely alone and add a controller (generally, I put all code related to 'users' within a users namespace, but for simplicity not in this article)

```ruby
# update the routes: config/routes.rb
Rails.application.routes.draw do
  resources :pets
  devise_for :users
  #    user url      rails path name      controller/action
  get  'login',      as: 'new_token',    to: 'tokens#new'
  post 'token',      as: 'tokens',       to: 'tokens#create'
  get 'auth/:token', as: 'token_auth',   to: 'tokens#auth'
  get 'logout',      as: 'token_delete', to: 'tokens#destroy'

  get 'landing',     as: 'landing',      to: 'landing#index'
  root to: 'landing#index'
end
```

---

# Token Authorization

```ruby
# touch app/controllers/tokens_controller.rb
class TokensController < ApplicationController
  skip_before_action :authenticate!, only: [:new, :create, :auth]
  def auth_token
    auth_token = params[:auth_token]
    user = GlobalID::Locator.locate_signed(auth_token, for: 'user_auth')
    if user.present?
      sign_in(user)
      flash[:notice] = "Welcome back! #{user.email}"
      redirect_to home_path
    else
      flash[:alert] = 'OOPS - something went wrong.'
      redirect_to root_path
    end
  end
  def destroy
    sign_out(current_user)
    redirect_to root_path
  end
end
```
---


# Setup Token Auth Mailer

Lets send the passwordless auth tokens via email

```ruby
bin/rails g mailer TokenAuth send_url

# app/mailers/token_auth_mailer.rb
class TokenAuthMailer < ApplicationMailer
  def send_url(user, auth_url)
    @user = user
    @url  = auth_url
    @host = Rails.application.config.hosts.first
    mail to: @user.email, subject: 'Sign in into #{@host}'
  end
end
```
---

# Email Contents

Lets include a greeting, the sending host and of course the auth_url (we will determine later)

```ruby
# app/views/token_auth_mailer/send_url.html.erb
<p>
  Hi <%= @user.email %>,
  For access to <%= @host %> <%= link_to "Click here", @url %>
</p>

# app/views/token_auth_mailer/send_url.text.erb
Hi <%= @user.email %>,
For access to <%= @host %> follow this link:
<%= @url %>
```

---

# Token URL Request Code

We have the logic and now we just need to tie it to the forms we create.

```ruby
touch app/controllers/tokens_controller.rb

# app/controllers/tokens_controller.rb
class TokensController < ApplicationController
  skip_before_action :authenticate!, only: [:new, :create, :auth]
  def new
    # I prefer to send local variables explicity instead of instance variables
    render :new, locals: {user: User.new}
  end
  def create
    user_params = params.require(:user).permit(:email)
    user = User.find_by(email: user_params[:email])
    if user
      global_id = user.to_sgid(expires_in: 1.hour, for: 'user_access')
      access_url = session_auth_url(token: global_id.to_s)
      TokenMailer.send_url(user, access_url).deliver_later
    else
      # do the same logic here to make time the same (prevent user mining)
    end
    redirect_to(root_path, notice: "Access-Link has been sent")
  end
end
```

---

# Token URL Request Form

As you can see from the code we expect the `email` attribute to be nested within a `user`

```ruby
touch app/views/users/tokens/new.html.erb

# app/views/users/tokens/new.html.erb
<%= form_for(user, class: "user", # params nested within :user for secure_params
                   url:   tokens_path, # give url to invoke when path and model differ
                   local: true, id: "login-form") do |form|  %>
  <div class="field">
    <%= form.label :email %>
    <%= form.text_field :email %>
  </div>

  <div class="actions">
    <%= form.submit %>
  </div>
<% end %>
```


---

# Appendix 4 - Devise w/ Token Strategy

Basic Setup

```ruby
rails new token_auth_devise_strategy_code
cd token_auth_devise_strategy_code

bin/rails db:create
bin/rails g controller Landing index
bin/rails g scaffold Pet name species

## config/routes.rb
Rails.application.routes.draw do
  resources :pets
  get 'home',    as: 'home',    to: 'pets#index'
  get 'landing', as: 'landing', to: 'landing#index'
  root to: "landing#index"
end
```

now the landing page and pets are freely available.

---

# Add Devise to add Authentication

```ruby
bundle add devise
bundle install
bin/rails generate devise:install
bin/rails generate devise User
# update the migration to match any added features
bin/rails db:migrate
```

---

# Basic Devise Config

```ruby
# config/environments/development.rb
self.default_url_options = { host: 'http://localhost:3000' }
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
# Rails.application.routes.default_url_options = { host: 'http://localhost:3000' }

# app/views/layouts/application.html.erb
<p class="notice"><%= notice %></p>
<p class="alert"><%= alert %></p>
```

---

# Basic Authentication Restrictions

Prevents all pages except 'landing', from non-authenticated access

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
end

# app/controllers/landing_controller.rb
class LandingController < ApplicationController
  skip_before_action :authenticate_user!
  def index
  end
end
```

---

# Email the User's Auth Token

Lets send the passwordless auth tokens via email

```ruby
bin/rails g mailer UserAuth send_link

# app/mailers/user_auth_mailer.rb
class UserAuthMailer < ApplicationMailer
  def send_link(user, url)
    @user = user
    @url  = url
    @host = Rails.application.config.hosts.first
    mail to: @user.email, subject: 'Sign in into #{@host}'
  end
end
```
---

# Email Contents

Lets include a greeting, the sending host and of course the auth_url (we will determine later)

```ruby
# app/views/user_auth_mailer/send_link.html.erb
<p>
  Hi <%= @user.email %>,
  For access to <%= @host %> <%= link_to "Click here", @url %>
</p>

# app/views/user_auth_mailer/send_link.text.erb
Hi <%= @user.email %>,
For access to <%= @host %> follow this link:
<%= @url %>
```

---

# Enable New Devise Auth Method

We need to extend Devise's sessions controller, based on our Secure Global ID knowledge

```ruby
rails g devise:controllers users -c=sessions

# app/controllers/users/sessions_controller.rb:
class Users::SessionsController < Devise::SessionsController
  def auth_token
    auth_token = params[:auth_token]
    user = GlobalID::Locator.locate_signed(auth_token, for: 'user_auth')
    if user.present?
      sign_in(user)
      flash[:notice] = "Welcome back! #{user.email}"
      redirect_to home_path
    else
      flash[:alert] = 'OOPS - something went wrong.'
      redirect_to root_path
    end
  end
end
```

---

# Code Devise Session Authorization

Tell rails (& Devise) about our newly extended Controller

```ruby
#config/routes.rb
Rails.application.routes.draw do
  resources :pets
  devise_for :users, controllers: { sessions: 'users/sessions' }
  devise_scope :user do  # the path to the NEW AUTHORIZATION based on Tokens
    get 'users/auth/:token', as: :auth_user_session, to: 'users/sessions#auth_token'
  end
  get 'home',    to: 'pets#index'
  get 'landing', to: 'landing#index'
  root           to: "landing#index"
end
```

---

# View Our Routes

We should now have:

```ruby
new_user_session      GET     /users/sign_in(.:format)   users/sessions#new
user_session          POST    /users/sign_in(.:format)   users/sessions#create
destroy_user_session  DELETE  /users/sign_out(.:format)  users/sessions#destroy
auth_user_session     GET     /users/auth(.:format)      users/sessions#auth_token
```

---

# Test the Session Authorization

```ruby
bin/rails c

user = User.first
auth_sgid = user.to_sgid(expires_in: 1.hour, for: 'user_access')
auth_token = auth_sgid.to_s
auth_url = Rails.application.routes.url_helpers
                .auth_user_session_url(login_token: auth_token)
# in a rails app use '.deliver_later' (to make emailing a background job)
UserAuthMailer.send_link(user, auth_url).deliver_now
```
copy this URL into the browser and you should now be on the 'pets' page

---

# Extend Devise with Token Auth

```ruby
mkdir app/lib/devise
mkdir app/lib/devise/models
mkdir app/lib/devise/strategies
touch app/lib/devise/models/token_authenticatable.rb
touch app/lib/devise/strategies/token_authenticatable.rb

# app/lib/devise/models/passwordless_authenticatable.rb
require Rails.root.join('app/lib/devise/strategies/token_authenticatable')
module Devise
  module Models
    module TokenAuthenticatable
      extend ActiveSupport::Concern
    end
  end
end
```

---

# Add A Token Auth Strategy

NOTE: this does not actually sign-someone in, it checks that the account is found and not locked and sends an authorized token.

```ruby
# app/lib/devise/strategies/token_authenticatable.rb
require 'devise/strategies/authenticatable'
require_relative '../../../mailers/user_mailer'

module Devise::Strategies
  class TokenAuthenticatable < Authenticatable
    def authenticate!
      email = params.dig(:user, :email)
      user = User.find_by(email: email)
      if user.present? && !user.locked_at? # and other restrictions as needed
        # this for setting MUST MATCH what's in the Auth Session Controller!
        auth_sgid = user.to_sgid(expires_in: 1.hour, for: 'user_access')
        auth_token = auth_sgid.to_s
        auth_url = Rails.application.routes.url_helpers
                        .auth_user_session_url(login_token: auth_token)
        UserAuthMailer.send_url(user, auth_url).deliver_later
      end
      fail!("An email was sent to you with an authorization link.")s
    end
  end
end
Warden::Strategies.add(:token_authenticatable, Devise::Strategies::TokenAuthenticatable)
```

---

# Load the Devise Strategy

The Devise initializer MUST load the strategy (at the top of the file)

```ruby
# config/initializers/devise.rb
Devise.add_module(:token_authenticatable, {
  strategy:   true,
  route:      :session,
  controller: :sessions,
  model:      'app/lib/devise/models/token_authenticatable'
})
```

Start rails to see if all paths are correct.

---

# Define the Strategy Usage

We need to update the User Model.

```ruby
class User < ApplicationRecord
  before_validation :set_password, on: :create
  # add other necessary Devise features (use only ONE strategy!)
  devise :token_authenticatable, :validatable
  # this will allow the session validations to be happy
  def password_required?
    false # because we aren't using passwords
  end
  private
  # since we aren't using passwords
  def set_password
    tmp_passwd = SecureRandom.alphanumeric(20)
    self.password = tmp_passwd
    self.password_confirmation = tmp_passwd
  end
end
```
Devise has Passwords deeply embedded so its way easier to disable them, than remove them!

NOTE: its easiest to use ONE strategy per model!
---

# Test Token Strategy Integration

If all is setup correctly the devise routes will still have:

```ruby
new_user_session      GET     /users/sign_in(.:format)   users/sessions#new
user_session          POST    /users/sign_in(.:format)   users/sessions#create
destroy_user_session  DELETE  /users/sign_out(.:format)  users/sessions#destroy
auth_user_session     GET     /users/auth(.:format)      users/sessions#auth_token
```

---

# Configure & Generate the Devise Views

We need to update `app/views/users/sessions/new.html.erb` and other forms for features we enabled so that:
1. configure them to be scoped (in the devise initializer)
2. remove the references to passwords in the forms
3. update the session_path references in the form from `session_path(resource_name)` to `user_session_path`

```ruby
# config/initializers/devise.rb:
config.scoped_views = true

# generate the devise views (to override them)
bin/rails generate devise:views users
```

---

# Override the Default Devise Views

Remember to:
* remove passwords
* update the session_path

```ruby
# app/views/users/sessions/new.html.erb
<h2>Log in</h2>
<%= form_for(resource, as: resource_name, url: user_session_path) do |f| %>
  <div class="field">
    <%= f.label :email %><br />
    <%= f.email_field :email, autofocus: true, autocomplete: "email" %>
  </div>
  <% if devise_mapping.rememberable? %>
    <div class="field">
      <%= f.check_box :remember_me %>
      <%= f.label :remember_me %>
    </div>
  <% end %>
  <div class="actions">
    <%= f.submit "Log in" %>
  </div>
<% end %>
<%= render "users/shared/links" %>
```

In our case, we only need to override `app/views/users/sessions/new.html.erb` since we haven't enabled anything but our strategy.

---

# Resources

* https://github.com/heartcombo/devise
* https://chelseatroy.com/2019/04/08/modifying-authentication-behavior-in-devise/
* http://blog.plataformatec.com.br/2019/01/custom-authentication-methods-with-devise/
* https://www.mintbit.com/blog/passwordless-authentication-in-ruby-on-rails-with-devise
* https://stackoverflow.com/questions/4223083/custom-authentication-strategy-for-devise
* https://github.com/heartcombo/devise/blob/main/app/controllers/devise/sessions_controller.rb
* https://github.com/heartcombo/devise/issues/1984 (answer from: josevalim commented on Jul 19, 2012 fixed views)
* https://stackoverflow.com/questions/25374187/creating-a-custom-devise-strategy (answer from: Srikanth Jeeva on Feb 2 '18, fixed strategy
* https://www.railsagency.com/blog/2016/06/15/how-to-create-custom-authentication-strategies-with-devise-and-warden/loading)
