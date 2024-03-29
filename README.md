# Devise gem basics

Devise gem https://github.com/heartcombo/devise

## Install

You can use template.rb
```
# for new apps
rails new blog -m https://raw.githubusercontent.com/duleorlovic/devise_gem_tips/main/template.rb

# for new apps but using local copy of template instead of github version
rails new blog -m ~/web-tips/devise_gem_tips/template.rb

# for existing apps
rails app:template LOCATION=https://raw.githubusercontent.com/duleorlovic/devise_gem_tips/main/template.rb

# for existing apps but using local copy of template instead of github version
rails app:template LOCATION=~/web-tips/devise_gem_tips/template.rb
```

You can read commands from the template and manually copy pase each one.
In template you can see the blocks like:

* Generate default user model if you do not already have users table
* Generate views, set mailer sender in credentials and add flash to layout
* Use [Const](https://github.com/duleorlovic/rails_helpers_and_const/blob/main/config/initializers/const.rb) helper
* Generate sample pages and protect ArticlesController using ApplicationUserController
* Sign in on development helper to sign in by GET request on staging and local
* Add controller and sytem test for signup
* Install importmap to add javascript_importmap_tags to layout and install
  stimulus and turbo
  You can notice that enabling turbo will break system sign up test

  ```
  rails test test/system/sign_up_test.rb:4

  NoMethodError in Devise::RegistrationsController#create
  undefined method `user_url' for #<Devise::RegistrationsController:0x0000000001bd00>
  ```
  so we need to patch devise

* Turbo and devise adding `turbo_stream` navigation format to
  `config/initializers/devise.rb`
  Flash messages are not seen so we need to add two elements `TurboFailureApp`
  and `TurboDeviseController`.

and this is the last thing for which we use template.rb

# API JWT Auth

For API Auth we will use https://github.com/waiting-for-dev/devise-jwt
and `template-for-api.rb`

```
rails app:template LOCATION=~/web-tips/devise_gem_tips/template-for-api.rb
```

Terminology:
* header: this can be used in native requests from mobile app for example
  `Authorization: Bearer my-jwt-token` so devise can determine current user.
* cookie: this is used inside browsers and also mobile apps can set in webview
  `CookieManager.getInstance().setCookie(BASE_URL, "oauth_token=${authToken}");`
  it has a limit of 4KB and it is just a plain string that server can read.
  Headers are used to set a cookie for example `curl -H "cookie:
  _gofordesi_webapp_session=asdasdasd"`

Note that inside mobile apps we need both authentications: for API and for
Webview.

Steps to enable authentication with jwt:

* Install devise-jwt
* Add devise_jwt_secret_key to credentials
* Configure devise config and User model
* Create JwtDenylist model and add `jwt_authenticatable` to User
* Show JWT token

Client can obtain JWT token in two ways, using html and using API.
To show Bearer jwt token on web you can use
```
# config/routes.rb
  get "show_jwt", controller: "application_user"
```
and
```
# app/controllers/application_user_controller.rb
  def show_jwt
    render json: { bearer_token: request.env['warden-jwt_auth.token'] }
  end
```

When user is logged in and navigate to `/show_jwt` response is
```
{
  bearer_token: "asd123"
}
```

Use existing controllers that are protected with
```
  before_action :authenticate_user!
```

Try in test
```
require 'devise/jwt/test_helpers'

user = User.last
headers = { 'Accept' => 'application/json', 'Content-Type' => 'application/json' }
auth_headers = Devise::JWT::TestHelpers.auth_headers(headers, user)
# {"Accept"=>"application/json",
# "Content-Type"=>"application/json",
# "Authorization"=>"Bearer eyJhbGciO..."}
get "articles", headers: auth_headers
```
for example in curl
```
export TOKEN=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI3YjgzNzUzMy0yN2JlLTRjZDQtYjNiMi1jMjY3M2UxZjQ1NjgiLCJzY3AiOiJ1c2VyIiwiYXVkIjpudWxsLCJpYXQiOjE2NTQxNTU1MDUsImV4cCI6MTY1NDE1OTEwNSwianRpIjoiOWY1OWUzODEtYzEwNy00YWMwLWI0YjMtMTQ5YjU3ODg5MzFmIn0.hB367AnlJhIaNjXAkSwWjszWYg8uRqDwtBgynSo36SQ

# list articles
curl -H "Authorization: Bearer $TOKEN" -H "Accept: application/json" -H "Content-Type: application/json" localhost:3000/articles

# create token
curl -XPOST -H "Authorization: Bearer $TOKEN" -H "Accept: application/json" -H "Content-Type: application/json" -d '{ "article": { "title": "my-title", "body": "my-body" } }' localhost:3000/articles

# delete token
curl -XDELETE -H "Authorization: Bearer $TOKEN" -H "Accept: application/json" -H "Content-Type: application/json" localhost:3000/articles/1
```
* on a POST we need to skip verify csrf token

Or you can enable cors (not sure why this helps)

```
bundle add rack-cors
```
and create a file
```
cat > config/initializers/cors.rb << 'HERE_DOC'
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'

    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
HERE_DOC
```


TODO: enable sign in and sign out api
TODO: test confirmation

require 'test_helper'

class MyConfirmationsControllerTest < ActionDispatch::IntegrationTest
  test 'sign_in after confirmation' do
    user = users(:unconfirmed)
    get user_confirmation_path(confirmation_token: user.confirmation_token)
    follow_redirect!
    assert_select "h4", "Basic Information"
  end
end

# Enable user log in with mobile phone

TODO:

## Errors

For error

```
ActionView::Template::Error: Devise could not find the `Warden::Proxy` instance on your request environment.
Make sure that your application is loading Devise and Warden as expected and that the `Warden::Manager` middleware is present in your middleware stack.
If you are seeing this on one of your tests, ensure that your tests are either executing the Rails middleware stack or that your tests are using the `Devise::Test::ControllerHelpers` module to inject the `request.env['warden']` object for you.
    app/controllers/application_controller.rb:143:in `current_user'
```

the problem is that background hotwire rendering does not know who is the
current_user so the solution it to pass as a parameter in places where you
render partials

```
# app/views/layouts/main_bell.html.erb
<%#
  user: required, use it instead current_user, since this is rendered from background
%>


# app/models/interest.rb
class Interest < ApplicationRecord
  after_update_commit(lambda do
    broadcast_replace_to(
      "main-bell-stream-for-profile-id-#{user_id}",
      target: 'main-bell',
      partial: 'layouts/main_bell',
      locals: {
        user: user
      }
    )
  end
end
```


## Test

Test authenticate_user

```
# test/test_helper.rb.rb
  # devise method: sign_in user
  include Devise::Test::IntegrationHelpers
```
and sign in user in tests (both in integration and system tests)
```
# test/controllers/articles_controller_test.rb
    sign_in users(:user)
```

For system tests, there is one for log in and reset password, and another for
register proccess (because register is updated more often).

