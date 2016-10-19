![](https://ga-dash.s3.amazonaws.com/production/assets/logo-9f88ae6c9c3871690e33280fcf557f33.png)

#Auth in Rails

### Why is this important?
<!-- framing the "why" in big-picture/real world examples -->
*This workshop is important because:*

Authenticating users is key to authorizing who is allows to do what in an application. We need to impliment these concepts in order to create and experience where users can have differentiated experiences on the application. For example, when we go to Feedly, we'd like to see new stories customized for each of our profiles, and when others go to gmail, we'd like them prevented from accessing our emails. 

### What are the objectives?
<!-- specific/measurable goal for students to achieve -->
*After this workshop, developers will be able to:*

* Implement an authentication system in Rails that securely stores users' passwords
* Build routes, controllers, and views necessary for a user to signup & login

### Where should we be now?
<!-- call out the skills that are prerequisites -->
*Before this workshop, developers should already be able to:*

* Illustrate the request/response cycle
* Compare and contrast sessions & cookies
* Build an MVC Rails application

## Authentication / Authorization

* **Authentication** verifies that a user is who they say they are. When a user logs into our site, we *authenticate* them by checking that the password they typed in matches the password we have stored for them.
* **Authorization** is the process of determining whether or not a user has *permission* to to perform certain actions on our site. For example, a user may *be authorized* to view their profile page and edit their own blog posts, but not to edit another user's blog posts.

A user must always first be authenticated, then it can be determined what they are authorized to do.

>Example: When Sarah enters a bar, a bouncer looks at her photo ID to ensure (authenticate) that she is who she claims. Sarah is thirty years old so she is allowed (authorized) to drink.


##Password Hashing

You've already seen [how not to store a password](https://www.youtube.com/watch?v=8ZtInClXe1Q).

In order to authenticate a user, we need to store their password in our database. This allows us to check that the user typed in the correct password when logging into our site.

The downside is that if anyone ever got access to our database, they would also have access to all of our users' login information. We use a [**hashing algorithm**](https://crackstation.net/hashing-security.htm#normalhashing) to avoid storing plain-text passwords in the database. We also use [**salt**](https://crackstation.net/hashing-security.htm#salt) to randomize the hashing algorithm, providing extra security against potential attacks. The plain-text password that has been hashed can be referred to as the **password digest**.

Think of a digested password as a firework. It is very easy to explode a firework (*hash plaintext into a digest*), but next to impossible to reverse that process (*turn the digest back into plaintext*). If I wanted to see if two sets of fireworks are the same (*a user is logging in, aka has provided their password and wishes to be authenticated*) we have to explode the fireworks again to compare it with the original explosion (*take the provided plaintext password, hash it again using the same algorithm, and match it with the saved password digest*).

![fireworks](http://i.giphy.com/122XXtx3oumxBm.gif)

## App Setup

Let's start a new Rails application:

* `rails new rails-auth -T -B -d postgresql`. (This command sets up a new application with no tests, no automatic bundle, and postgres as the database.)
* `cd rails-auth`
* `rake db:create`
* `subl .`

The library of choice for password hashing is `BCrypt`, which we will add to our gemfile. In Rails, the convention is to add all our business logic into the models, so we will be writing most of our code in the `User` model.

Remember, remember: **never store plaintext passwords**, only the digested versions. 

Let's uncomment `bcrypt` at the bottom of our `Gemfile` as we will need it to digest (hash) the plain-text password and store it in a `password_digest` field of our database's `users` table.

`Gemfile`

```ruby
# Use ActiveModel has_secure_password
gem 'bcrypt', '~> 3.1.7'
```

Then run `bundle` to install `bcrypt` and the other gems.

### Playing With `BCrypt`

As soon as something is installed via bundler we can access it via our `rails console.` Let's play in console.

```bash
Loading development environment (Rails 4.1.6)
## Let's create our first password & save the hashed output to a variable
2.1.0 :001 > hashed_pass = BCrypt::Password.create("swordfish")
=> "$2a$10$6MQQCxBpfu16koDVs3zkbeSXn1z4fqKx9xLp4.UOBQBDkgFaukWM2"

## Let's compare our password to another
2.1.0 :003 > BCrypt::Password.new(hashed_pass) == "tunafish"
=> false
 	
## Let's compare our password to original
2.1.0 :004 > BCrypt::Password.new(hashed_pass) == "swordfish"
=> true
 	
## Exit
2.1.0 :005 > exit
```

> Note: the `==` method for `BCrypt::Password` is different than the typical comparator in Ruby say for an `Object`.

```ruby
BCrypt::Password.instance_method(:==) == String.instance_method(:==)
=> false
```

How will Bcypts `==` help us **authenticate** a `User`?

[BCrypt](https://en.wikipedia.org/wiki/Bcrypt) uses ["salt"](https://en.wikipedia.org/wiki/Salt_(cryptography)) to protect against [rainbow table](https://en.wikipedia.org/wiki/Rainbow_table) attacks and is an [adaptive function](https://codiscope.com/cryptographic-hash-functions/) (see section: "Adaptive Hash Functions") to protect against [brute-force](https://en.wikipedia.org/wiki/Brute-force_search) attacks.

## Test Setup

* Add the rspec gem to both test and development environments, then run `bundle`

```ruby
group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger console
  gem 'pry-byebug'

  # Rspec allows us to write tests for rails
  gem 'rspec-rails'
end
```

`bundle` and then run the command `rails g rspec:install` to initialize rspec as your testing suite. Now a `spec` directory has been created. Additionally, rspec will automatically generate tests for any files created by the `rails generate` command.

## Model Setup

Let's leave our controllers be for the time being and setup our models. Our tests depend on a `User` model existing. The default attribute type is string, if we don't specify.

```bash
rails g model user email password_digest
```

`email` is the natural username for our user, and the `password_digest` is where we'll store the user's hashed password.

>If you ever make a mistake during a generation, you can reverse it with `rails destroy <resourceType> <resourceName>`. In this case it would simply be `rails d model user`.

Great, let's run the migrations!

```bash
rake db:migrate
```

Now we can ensure we build our `User` model to specifications by passing some tests we've been given!

Inside the directory `spec` overwrite the existing file `/models/user_spec.rb` with the below tests.

```ruby
require "rails_helper"

describe User, type: :model do

  it "can create a new user" do
    expect(User.new).to be_a User
  end

  context 'Initialization' do
    subject(:user) { User.new }

    it "allows the getting of a password" do
      expect(user).to respond_to(:password)
    end

    it "allows the setting of a password" do
      expect(user).to respond_to(:password=).with(1).argument
    end

    it "creates a password digest when a password has been set" do
      #password digest starts as nil
      expect(user.password_digest).to be_nil
      #password is set
      user.password = "swordfish"
      #password digest is created after password is set
      expect(user.password_digest).not_to be_nil
    end
    it "ensures the password digest is not the password" do
      user.password = "swordfish"
      expect(user.password_digest).not_to eq(user.password)
    end
  end

  context 'Validation' do
    subject(:user) do
      #create a user in active memory
      User.new({
        email: "bana@na.com",
        password: "adsf1234",
        password_confirmation: "adsf1234"
      })
    end
    it "validates presence of password_digest" do
      #clear values of password & password_confirmation
      user.password_digest = nil
      expect(user).not_to be_valid
    end

    it "validates presence of email" do
      #clear values of email
      user.email = nil
      expect(user).not_to be_valid
    end

    it "validates password & password confirmation match" do
      user.password_confirmation = "not the same"
      expect(user).not_to be_valid
    end
  end

  context 'Authentication' do
    before(:all) do
      #clear all users
      User.destroy_all
      #save a user to the database
      @user = User.create({
        email: "shmee@me.com",
        password: "jumanji",
        password_confirmation: "jumanji"
      })
    end
    it "restricts passwords from saving to the db" do
      found_user = User.all.first
      expect(found_user.password).to eq(nil)
    end

    describe "#authenticate" do
      it "returns the user when the correct password is provided" do
        expect(@user.authenticate("jumanji")).to eq(@user)
      end

      it "returns false when an incorrect password is provided" do
        expect(@user.authenticate("ijnamuj")).to eq(false)
      end
    end

    describe "::confirm" do
      it "checks if a specified user & password combination exists" do
        user_email = "shmee@me.com"
        user_password = "jumanji"
        found_user = User.find_by_email(user_email)
        expect(User.confirm(user_email, user_password)).to eq(found_user.authenticate(user_password))
      end
    end
  end
end
```

Run them with:

```bash
rspec
```

## Authentication (TDD Style)

In the process of passing these tests we will build all the logic for an authentication system! You should never have to write this code from scratch, but it is very important you understand what is going on.

>Exercise: Think, pair, share on what the below code is doing (7 minutes).

```ruby
class User < ActiveRecord::Base
  BCrypt::Engine.cost = 12

  # email & password_digest fields must exist
  validates :email, :password_digest, presence: true
  # a user must have a password & password confirmation field
  # the fields are match against each other but never persisted to the database
  validates_confirmation_of :password
  # TODO: add validator for unique emails
  
  def password=(unencrypted_password)
    #raise scope of password to instance
    @password = unencrypted_password
    self.password_digest = BCrypt::Password.create(@password)
  end

  def password
    #get password, equivalent to `attr_reader :password`
    @password
  end

  # to authenticate the user using bcrypt's built in 
  def authenticate(unencrypted_password)
    # check that a hashed version of the unencrypted password is the same as the secure password
    BCrypt::Password.new(self.password_digest) == unencrypted_password ? self : false
  end

  # class method `::confirm`
  def self.confirm(email_param, password_param)
    # add a unique email validator later
    user = User.find_by_email(email_param)
    user.authenticate(password_param)
  end

end
```

###Challenge: Unique emails

Write a test that validates the uniquness of emails, watch it fail, then write the code that passes it.

## Routes, Controllers, & Views for Signup

###User stories...

A user should be able to... 


* go to `/signup` and have the application execute the `users#new` action to render `/views/users/new.html.erb`.

* see a `form_for` on `users#new` that displays email, password, and password_confirmation data.

* submit the form on `users#new` to `users#create` which creates a new user, logs them in, and redirects to `user#show`.

* go to `/users/:id/` and see their profile page.

Let's get started!

## Routes

Let's edit our `config/routes.rb` file...


```ruby
Rails.application.routes.draw do

  root to: "welcome#index"

  get "/login", to: "sessions#new"

  post "/sessions", to: "sessions#create"

  get "/sign_up", to: "users#new", as: "sign_up"

  resources :users

end
```

If you haven't seen it before, `resources` will auto generate all the RESTful routes for a `User`.

Run `rake routes` to see all the application's routes.

## Home Page

**Challenge:** Start your application and pass these user stories. On the `root_path`:

* A user can see a welcome message`
* A user can click a "Sign Up" button that directs them to the `sign_up_path`

## Controllers

* Let's create `UsersController` with the command: `rails g controller users new create show`

* Let's add a private method that creates strong parameters for specific attributes of the user

You should end up something along the lines of...

```ruby
class UsersController < ApplicationController
  
  def new
  end

  def create
    # TODO: once the controller is implemented don't forget to also sign the user in
  end

  def show
  end

  private
  
  def user_params
    params.require(:user).permit(:email, :password, :password_confirmation)
  end

end
```

##Challenge: Implement Signup

####Step 1

* For your `/sign_up` route, which hits the action `users#new`, render a file `new.html.erb` in `/views/users`.

####Step 2

* In that view add a `form_for` referencing user; have it post to `users#create` with `email`, `password` and `password_confirmation`.

####Step 3

* Create the user in `users#create` and when done have it redirect to `user#show` (later we will have them also be logged-in in this step)
	* Tip: create a condition that checks if the user was saved correctly. Hint: first build the user in memory with `.new` then check `if @user.save` proceed as normal `else` render the signup page again.

####Step 4

* `users#show` will find the [current user](#current_user) and display their profile page

##Session Management

<h3 id="session_creation">Login</h3>

Since creating a session is essentially what we mean when want to login, and logging out is destroying a session. We have a single controller dedicated to session management, `SessionsController`.

`app/controllers/sessions_controller.rb`

```ruby
class SessionsController < ApplicationController
  def new
    #TODO: render a login view
  end

  def create
    #call the User#confirm method
    if User.confirm(params[:email], params[:password])
      # this creates the session, logging in the user
      session[:user_id] = user.id
      #redirect to the show page
      redirect_to user_path(user.id)
    else
      #there was an error logging the user in
      redirect_to login_path
    end
  end
  
  def destroy
    #TODO: logout the current user
  end
  
end

```
After we authenticate someone we set `session[:user_id] = user.id`. This allows the `user.id` to be stored in a cookie for lookup later.

Now that we know how to login a user with `session[:user_id] = user.id` let's also make sure to do that when a user is signed up (it is good UX for a signup to automatically perform a login).

Tip: Try running `rake notes` to see all the items that have been marked as `TODO` in the comments.

<h3 id="current_user">Current user</h3>

Since we need to authenticate each request and to do so we have to read the `user_id` out of the `session` object, let's consider making a few helper methods to do so.

A login for a user is when we set a unique identifier for a user in their session, aka `session[:user_id] = user.id`, so they are able to maintain their logged-in state by sending that unique piece of data back to us each time they send a new request. What is we could have a helper method that does this for us and caches the value of the  `current_user` for the duration of each request?

```ruby
class ApplicationController < ActionController::Base
  def current_user
    @current_user ||= session[:user_id] && User.find_by_id(session[:user_id])
  end
  helper_method :current_user #make it available in views
end
```

>Who can explain how the `current_user` method works?

The above method defines `@current_user` if it is not already defined. The way the `&&` operator works is that it will keep evaluating if `session[:user_id]` is defined and then set `@current_user` to whatever the *last item* evaluated is; in this case it would be `User.find_by_id(session[:user_id])`, so the user itself.

>Prefer `find_by_id(<id>)` instead of `find(<id>)`, as it does not rails an exception, but instead returns `nil` if the id is invalid.

The method `current_user` in is very useful for:

* **Conditional views** based on the `current_user`'s state
	* I.e. is a login or logout button displayed in the nav_bar?
* **Authorization** to CRUD resources
	* I.e. determine if `current_user` is accessing their own or another's information.

<h3 id="logout">Logout</h3>

In `session#destroy`, delete the key `:user_id`, clear the `@current_user`, and redirect to your `root_path`.

```ruby
session.delete(:user_id)
@current_user = nil
```

## More Notes

###Refactor

* Using [`has_secure_password`](http://api.rubyonrails.org/classes/ActiveModel/SecurePassword/ClassMethods.html#method-i-has_secure_password) can magically refactor a lot of our password storing logic in the User model. Try it out and see if the tests still pass...

![success!](http://i.giphy.com/b6oC7bEdJD26c.gif)


###Authorization

You can secure your routes with a `:before_action` [filter](http://guides.rubyonrails.org/action_controller_overview.html#filters). This code can be run before any `controller#action` in the application. For example let's say a user must be logged before they can see all the `posts` in the application. You could create a private method in the application controller with a name such as `require_login`.

application_controller.rb

```ruby
class ApplicationController < ActionController::Base
 
  private
 
  def require_login
    if !current_user
      #send an http response to stop program execution
      redirect_to root_path
    end
  end
end
```

>Note: One could also use `unless` in lieu of `if !`

Now use a `before_action` to run the `require_login` method before any actions the `PostController` will perform.

posts_controller.rb

```ruby
class PostController < ApplicationController
  before_action :require_login
  def index
  end
end
```

Further define which actions this hook is applied to with the `only` & `except` [options](http://guides.rubyonrails.org/action_controller_overview.html#highlighter_858492).

<h3 id="flash_msgs">Bonus: Adding Flash Messages</h3>

We want to notify users of any errors. Rails patterns use a [flash hash](http://guides.rubyonrails.org/action_controller_overview.html#the-flash) to do so.

session_controller.rb

```ruby
class SessionsController < ApplicationController
  def new
  end

  def create
    user_params = params.require(:user)
    user = User.confirm(user_params[:email], user_params[:password])
    if user
      # use our handy login method
      login(user)
      redirect_to user_path(user.id)
    else
       # Flash an error message
      flash[:error] = "Failed To Authenticate. Please try again."
      redirect_to "/login"
    end

  end
end
```

The [twitter-bootstrap-rails](https://github.com/seyhunak/twitter-bootstrap-rails) gem has a `bootstrap_flash` that adds some nice styling to the flash messages. Require the library in project with `rails generate bootstrap:install static`

We can then render these message and style them with a class that matches their name on all pages.

application.html.erb

```html+erb
<!-- include this just above the yield -->
<%= bootstrap_flash %>

<%= yield %>

```

Now you need to pass in flash messages to your views! For example my user controller could look like this.

user_controller.rb

```ruby
class UsersController < ApplicationController
  before_action :require_login, only: :index

  # to illustrate a before_action
  def index
    @users = User.all
  end

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)
    if @user.save
      #login user
      session[:user_id] = @user.id
      #redirect to user#show w/ success message
      redirect_to @user, flash: { success: "Successfully signed up!" }
    else
      #there was an error, go back to signup page & display message
      redirect_to sign_up_path, flash: { error: @user.errors.full_messages.to_sentence }
    end
  end

  def show
    @user = current_user
  end

  private

  def user_params
    params.require(:user).permit(:email, :password, :password_confirmation)
  end
end
```

##Closing Thoughts

* What is the difference between authorization and authentication?
* How should we not store passwords?
* Why is BCrypt useful and how do we use it to authenticate a user
* What does it mean for a user to be "logged in"?

##Additional Resources

* [Rails Tutorial: login & logout](https://www.railstutorial.org/book/log_in_log_out)
* [Devise](https://github.com/plataformatec/devise)
* [How public-key cryptosystems work](https://www.youtube.com/watch?v=3QnD2c4Xovk)
