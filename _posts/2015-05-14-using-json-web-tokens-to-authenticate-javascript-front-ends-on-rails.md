---
layout: post
title: "Using JSON Web Tokens to Authenticate JavaScript Front-Ends on Rails"
tags:
  - ruby
  - rails
  - javascript
---

While working on a project recently, I encountered a problem I haven't had to
tangle with in a while: authenticating front-end applications against a Rails
API. The last time I was even dabbling in this realm, jQuery was everything,
CORS was still in its infancy, and JSONP was still a thing (that's not a thing
anymore, right?). The only way I ever managed to scrape by in this hostile
environment was to let Rails' asset pipeline serve up the front-end app and
rely on same-origin requests and regular ol' cookies to handle authentication.
I didn't like it, but I survived. Eventually, I got away from front-end
concerns almost completely.

Since those dark times, a few tools have cropped up and improved the landscape.
As I mentioned before, I have recently been working on a project that I wanted
to have a stand-alone front-end app for (inb4 single page apps don't work). I
wanted to avoid the messes I've encountered with JavaScript-heavy Rails apps
and keep Rails as API-centered as possible. Once I had a functioning back-end,
I knocked together a rudimentary front-end with React using Bower to manage
dependencies and Jekyll to compile it all down to a static page.

At first, I got started by symlinking the front-end to _public/_ in my Rails
app and setting `protect_from_forgery with: :null_session`. That was good
enough for me to get my feet wet with React and get back into the swing of
things with JavaScript (with which I hadn't really done anything of consequence
with since before ES5). However, this setup was clearly deficient for even a
development deploy to Heroku. When I got to that point, I had to give thought
to where I'd host the front-end and how I'd manage authentication.

There were a few other, related problems to solve, but I want to focus on how
I did authentication for the rest of this post. I had a few characteristics
that I wanted to satisfy:

* Let Devise do the heavy lifting
* Not require the front-end and back-ends to be on the same hostname
* Not clutter my application code with a bunch of (what should be standardized)
  authentication logic
* Leave open the possibility of separating the app into microservices,
  including an authentication service

To that end, I scrapped together a handful of tools, blog posts, and sites that
have already solved this problem. In that bucket was, of course, Devise, and
JSON Web Tokens, or JWTs as I'll refer to them hereafter. A really nice [blog post
by Andrea Pavoni][nebulab-post] was a valuable resource, but I had a few
disagreements with their implementation. Heroku does a really good job with
their dashboard app, and as a bonus they already have a separate ID service, so
I knew that however they were doing it was probably something I should pay
attention to.

I wanted to stick with Devise for a couple reasons. One: I'm not a big fan of
Building It Here™.  I'm trying to build a workflow automation service, not
pioneer user authentication. Two: Devise is battle tested. There are people
much smarter than me with a log of 2,804 closed issues (as of writing). Plenty
of those pertain to security defects that have been identified, fixed, and I
have no idea about. I probably shouldn't recreate each of those defects on my
own.

JWT was vaguely on my radar after it came up during the Q&A of talk by Lance
Gleason at the Atlanta Ruby Users Group. They are pretty neat, but nothing too
special. Essentially, they are an encrypted JSON object that can contain
arbitrary claims, like a user id. The entire claim object can be decrypted and
inspected by anyone with the secret key. This is useful, because instead of storing
a session token server-side, looking it up, and verifying it every request, the
token can be verified by anyone with the secret. Since the claims are arbitrary,
you could store a user's name or any other information in there. That's great for
distributed systems where not every service needs to have access to the "truth"
for user records. Some services may just need the name for display purposes.
Because it's encrypted, it's safe from being tampered with by the user. A downside
is that you can't easily revoke a token. You could blacklist them, but then you
lose all the benefits of not needing to store and verify them server-side. There
are several ways to do this, but I think the easiest is to keep your session
TTLs short and make token re-issuance transparent.

To issue tokens, I used the [JWT ruby gem][jwt-ruby]. I wrote a thin wrapper around
it to encapsulate whatever application-specific stuff I needed: basically the secret
and TTL:

```ruby
require 'jwt'

class JsonWebToken
  def self.encode(payload, expiration = 24.hours.from_now)
    payload = payload.dup
    payload['exp'] = expiration.to_i
    JWT.encode(payload, Rails.application.secrets.json_web_token_secret)
  end

  def self.decode(token)
    JWT.decode(token, Rails.application.secrets.json_web_token_secret).first
  end
end
```

I wanted to imitate Heroku's ID service with my Rails app. I didn't want my
front-end handling usernames and passwords at any point. I just wanted to
redirect the user to the back-end to sign in, but then be able to acquire
session tokens thereafter. To that end, I left the Devise sign-in/-up
controllers and forms all as the normally are on the back-end, and created a
session tokens resource for issuing a JWT for the `current_user`:

```ruby
require 'json_web_token'

class SessionTokensController < ApplicationController
  before_action :authenticate_user!
  skip_before_action :verify_authenticity_token

  def create
    token = JsonWebToken.encode('user_id' => current_user.id)

    respond_to do |format|
      format.json { render json: {'token' => token}, status: :created }
    end
  end

  def show
    respond_to do |format|
      format.json { render json: {'logged_in' => true} }
    end
  end
end
```

So far, my approach was very similar to that of Nebulab's, but here's where
things start to diverge. I didn't want all the authentication goop cluttering
up my `ApplicationController`. Since I'm using Devise, I'm already using Warden.
Warden allows you to specify a cascade of [strategies][warden-strategies] for
authenticating a request.  By default, Devise uses the username/password
strategy, but you can layer multiple together. I just wanted to add another
strategy to the stack.

```ruby
require 'json_web_token'

module Devise
  module Strategies
    class JsonWebToken < Base
      def valid?
        !request.headers['Authorization'].nil?
      end

      def authenticate!
        if claims and user = User.find_by_id(claims.fetch('user_id'))
          success! user
        else
          fail!
        end
      end

      private

      def claims
        auth_header = request.headers['Authorization'] and
          token = auth_header.split(' ').last and
          ::JsonWebToken.decode(token)
      rescue
        nil
      end
    end
  end
end
```

Defining the `valid?` method is optional, but allows you to determine whether
the strategy should be used for any request. My implementation means that this
strategy will only be active for requests with an `Authorization` HTTP header.
The `authenticate!` method decrypts the JWT, and finds the corresponding user.
The result is that without any mess in the application code, we end up with
a `current_user`, just like you're used to, any time a request with a valid JWT
comes in.

To inject the above strategy into the stack you need to add this bit to the
_devise.rb_ initializer:

```ruby
config.warden do |manager|
  manager.strategies.add(:jwt, Devise::Strategies::JsonWebToken)
  manager.default_strategies(scope: :user).unshift :jwt
end
```

To support cross-site requests, I used [Rack CORS][rack-cors]. This enabled the front-end to
make, requests for session tokens, and then later to make requests for the
resources, from domains other than the one hosting the API.

The next part was making the front-end request and use JWTs. I put together
a `Session` object to use across the front-end. It has a few features. It can
refresh a JWT by making a POST request to the session tokens resource using a
regular ol' cookie (normal Devise-auth'd ajax at that point). It stores JWTs
in `localStorage`. It can check the status of a token, and it can delete the
token from `localStorage` and send the browser to Devise's sign-out endpoint.
I'm slightly dissatisfied with this part. It makes a GET request to log you
out, but it seems like that's what Heroku uses, so I'll go with it for now.

```javascript
var Session = function () {
  this.authenticated = function () {
    var begin = Promise.resolve();

    if (this.getSessionToken() === undefined) {
      begin = begin.then(this.refreshSessionToken);
    }

    return begin.then(function () {
      var options = {
        headers: new Headers({
          'Authorization': 'Bearer ' + this.getSessionToken(),
          'Content-Type': 'application/json',
          'Accept': 'application/json'
        })
      };
      return window.fetch(Urls.sessionTokenUrl, options).then(function (response) {
        if (response.status === 401) {
          return false;
        } else {
          return true;
        }
      });
    }.bind(this));
  };

  this.deAuthenticate = function () {
    delete localStorage.sessionToken;
    window.location = Urls.signOutUrl;
  }

  this.refreshSessionToken = function () {
    var options = {
      credentials: 'include',
      headers: new Headers({
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      }),
      method: 'POST'
    };

    return window.fetch(Urls.sessionTokenUrl, options).
      then(function (response) {
        if (response.status === 401) {
          throw new RequestFailedException('not signed in');
        }
        return response.json().then(function (data) {
          localStorage.sessionToken = data.token;
        });
      });
  };

  this.getSessionToken = function () {
    return localStorage.sessionToken;
  }
};

var session = new Session();
```

To enable the sign-out via GET in devise, add this to the _devise.rb_
initializer:

```ruby
config.sign_out_via = :get
```

To make use of these tokens, I wrote a `Controller` base object, which I use as
a prototype for all my resource controllers. It's essentially a wrapper around
`GlobalFetch.fetch()` that makes sure to include the `Authorization` header in
requests. If a request fails with a 401, it will refresh the token and attempt
to retry the request.

```javascript
var Controller = function () {
  var MAX_RETRIES = 1;

  this.fetch = function (input, init) {
    var _fetch = function (tries) {
      if (init.headers === undefined) {
        init.headers = new Headers();
      }

      init.headers.set('Accept', 'application/json');
      init.headers.set('Authorization', 'Bearer ' + session.getSessionToken());
      init.headers.set('Content-Type', 'application/json');

      return fetch(input, init).then(function (response) {
        if (response.status !== 401) {
          return response;
        } else {
          if (tries >= MAX_RETRIES) {
            throw new RequestFailedException('too many retries');
          }

          return session.refreshSessionToken().then(function () {
            return _fetch(tries + 1);
          });
        }
      }.bind(this));
    };

    return _fetch(0);
  };
};
```

The last bit of polish I needed for signing in was a link to the sign in page
of Devise and a little code to redirect back to the front-end appropriately.
This is literally the whole `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base
  protect_from_forgery with: :null_session
  respond_to :html, :json
  after_action :store_location

  def store_location
    case request.path
    when '/users/sign_in', '/users/sign_up'
      session[:return_to] = request.referer
    end
  end

  def after_sign_in_path_for(resource)
    session.delete(:return_to) || root_path
  end
end
```

I'm not terribly pleased about this part (add it to the list containing
sign-out via GET), but it gets the job done, and fairly reliably too.

The result of all this is that my back-end supports tried-and-true sign up and
sign in via Devise, can issue expirable session tokens, and subsequently
authenticate a user via such a token. The front-end can acquire such a token
once a standard, cookie-backed session has been established. Using the token,
it can then make requests against protected resources of the API. If a token
expires, it will transparently refresh and should be completely unnoticeable to
users. The app now works without needing the front-end and the back-end to be
so tightly coupled. Additionally, it would not be a huge stretch to segment the
app into different services that could independently verify the JWTs.

[nebulab-post]: http://nebulab.it/blog/authentication-with-rails-jwt-and-react
[jwt-ruby]: https://github.com/progrium/ruby-jwt
[warden-strategies]: https://github.com/hassox/warden/wiki/Strategies
[rack-cors]: https://github.com/cyu/rack-cors
