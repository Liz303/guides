With API-only applications gaining popularity and  Rails 5 being just around the corner, the most ubiquitous methods of authentication are now becoming token-based. In this tutorial, a short overview of token-based authentication will be given and implemented into a Rails 5 API-only application.


## What is token-based authentication?
 
 Token-based authentication (also known as [JSON Web Token authentication](https://jwt.io/)) is a new way of handling authentication of users in applications. It is an alternative to session-based authentication. The most notable difference between the two is that session-based authentication is persisted server-side i.e a record is created for each logged-in user. Token-based authentication is stateless - it does not store anything on the server, but creates a unique encoded token that is checked every time a request is made.  

### Benefits of token-based authentication
There are several benefits to using such approach. The main ones are:
- Cross-domain / CORS: cookies + CORS don't play well across different domains. A token-based approach allows you to make AJAX calls to any server, on any domain because you use an HTTP header to transmit the user information.

- Stateless:Tokens are stateless there is no need to keep a session store, the token is a self-contanined entity that contains all the user information in it. 
- Decoupling: You are not tied to a particular authentication scheme. The token might be generated anywhere, hence the API can be called from anywhere with a single way of authenticating all the calls.

- Mobile ready: Cookies are a problem when it comes to storing user information on native mobile applications. Adopting a token-based approach simplifies the process significantly.

- CSRF: Because the application does not rely on cookies for authentication, it is invulnerable cross site request attacks.

- Performance: In terms of server-side load, a network roundtrip (e.g. finding a session on database) is likely to take more time than calculating an HMACSHA256 code to validate a token and parsing its contents.

### How does it work?
The way token-based authentication works is simple: The user enters his/her credentials and sends a request to the server. If the credentials are correct, the server creates a unique HMACSHA256 encoded token also known as JSON web token (JWT). The client stores the JWT and makes all subsequent requests to the server with the token attached. The server authenticates the user by comparing the JWT sent with the request to the one it has stored in the database. Here is a simple diagram of the process:

![token-based auth](http://i.imgur.com/xkvip2y.jpg)

### What does a JWT token contain?
First Column Header | Second Column Header | Third Column
------------------- | -------------------- | ------------
Content from cell 1 | | Content from cell 3
Another cell 1 | Another cell 2

## Setting up a token-based authentication with Rails 5

Enought theory, it's time for practice. The first step is to create a new Rails 5 API-only application:

```bash
rails _5.0.0.beta3_ new api-app --api
```


Using `--api-only` when create an  API-only application. [API-only](http://edgeguides.rubyonrails.org/api_app.html) applications are a new feature that is included in Rails 5. They are trimmed down versions of standard Rails applications that have features like `.erb` views, helpers and assets removed because of redundancy. They come with special middlewares such as `ActionController::API` , request throttling, easily configurable CORS and other custom-waived features for building APIs.

To make set up token authentication for the newly created application, several steps have to be taken:

* A model that can be accessed
* A way of encoding and decoding JWT tokens
* Methods for checking if the user is authenticated.
* Controllers for creating and logging in users.
* Routes for creating users and logging them in and out.

### Creating the user model
 First, the user model must be created:
```bash
 rails g model User name email password_digest
```
 Run the migrations:
```bash
 rails db:migrate
```
 By running these methods, we create a user model in with name, e-mail and password fields and have its schema migrated in the database.
 
The method `has_secure_password`  has to be added to the model to make sure the password is properly encrypted into the database:
 `has_secure_password`  is part of the bcrypt gem, so we have to install it first. Add it to the gemfile:
 ```ruby
 #Gemfile.rb
 gem 'bcrypt', '~> 3.1.7'
 ```
 And install it:
 ```bash
  bundle install
 ```
 With the gem installed, the method can be included in the model:
 ```ruby
#app/user.rb
 
class User < ApplicationRecord
  has_secure_password
end

```
### Encoding and decoding JWT tokens

Once the user model is done, the implementation of the JWT token generation can start. First, the `jwt` gem will make encoding and decoding of HMACSHA256 tokens available in the Rails application. Let's install it:

 
 ```ruby
  gem 'jwt'

```
And install it:

```bash
bundle install

```
Once the  gem is installed, it can be accessed through the `JWT` global variable. Because the methods have to be encapsulated somehow, a singleton class is  a great way of wrapping the methods and using them in other constructs:

 ```ruby
 
 # lib/json_web_token.rb

class JsonWebToken
  class << self
    def encode(payload, exp = 24.hours.from_now)
      payload[:exp] = exp.to_i
      JWT.encode(payload, Rails.application.secrets.secret_key_base)
    end

    def decode(token)
      body = JWT.decode(token, Rails.application.secrets.secret_key_base)[0]
      HashWithIndifferentAccess.new body
    rescue
      nil
    end
  end
end
```
The first method,`encode`, includes three parameters - takes the user id, the expiration time (1 day) and the unique secret key of your rails application in order to create a key. The second method,`decode` takes the token and uses the application's secret key to decode it. These are the two cases in which these methods will be used:
 * For authenticating the user and generate a token for him/her using `encode`
 * To check if the user's token appended in each request is correct by using `decode`. 

To make sure everything will work, we have to load the contents of the `lib` directory when the Rails applciation loads:
```ruby
    #config/application.rb
module ApiApp
  class Application < Rails::Application
    #.....
    config.autoload_paths << Rails.root.join('lib')
    #.....
    end
   end
```
### Authenticating users
 Instead of using private controller methods, we will use [simple_command](https://github.com/nebulab/simple_command).

The simple command gem is an easy way of creating services. Its role is similar to helpers, but instead of facilitating the connection between the controller and the view, it does the same for the controller and the model. In this way, the code in the models and controllers is reduced to minimum.

Add the gem to your `Gemfile`:
```ruby
gem 'simple_command'
```
And bundle it:
```bash
bundle install
```
Then, the alias methods of the `simple_command` can be easiy used in a class by writing `prepend SimpleCommand`. Here is how a command is structured:
```ruby

class AuthenticateUser
   prepend SimpleCommand

  def initialize()
   #this is where parameters are taken when the command is called
  end

  def call
   #this is where the result gets returned
  end

end
```
The command has to take the user's e-mail and password and return the user if the credentials are correct. Here is how this can be done:
```ruby
# app/commands/authenticate_user.rb

class AuthenticateUser
  prepend SimpleCommand

  def initialize(email, password)
    @email = email
    @password = password
  end

  def call
    JsonWebToken.encode(user_id: user.id) if user
  end

  private

  attr_accessor :email, :password

  def user
    user = User.find_by_email(email)
    return user if user && user.authenticate(password)

    errors.add :user_authentication, 'invalid credentials'
    nil
  end
end
```
The command takes the parameters and initializes a class instance with `email` and `passowrd` attributes that are accessible wthing the class. The private method `user` uses the credentials to check if the user exists in the database using `User.find_by_email` . If the user is found, it uses the built-in `authenticate` method (available by putting [has_secure_password](http://api.rubyonrails.org/classes/ActiveModel/SecurePassword/ClassMethods.html) in the User model to check if the user's password is correct. If everything is true, the user will be returned. If not, the method will return `nil`.

### Checking if user is authorized
The token creation is done, but there is no way to check if a token appended to request is valid. The command for authorization has to take the `headers` of the request and decode the token using the `decode` method in the `JsonWebToken` singleton. 


> **A refresher on headers**
  Each http request has fields known as [headers](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html). Headers contain informationabout the request that can be helpful to the server such as the format of the request body. Tokens are usually attached to the 'Authorization' header.
  
  Here is how the code is structured:
```ruby
# app/commands/check_if_user_authorized.rb

class AuthorizeApiRequest
  prepend SimpleCommand

  def initialize(headers = {})
    @headers = headers
  end

  def call
    user
  end

  private

  attr_reader :headers

  def user
    @user ||= User.find(decoded_auth_token[:user_id]) if decoded_auth_token
    @user || errors.add(:token, 'Invalid token') && nil
  end

  def decoded_auth_token
    @decoded_auth_token ||= JsonWebToken.decode(http_auth_header)
  end

  def http_auth_header
    if headers['Authorization'].present?
      return headers['Authorization'].split(' ').last
    else
      errors.add(:token, 'Missing token')
    end
    nil
  end
end
```
There is a chain of methods getting executed. Let's start from bottom to top:

The last method in the chain, `http_auth_header`, extracts the token from the authorization header received in the initialization of the class the class.
. The second method in the chain is `decoded_auth_token` which decodes the token received from `http_auth_header`to get get the id of the user which is contained in the decoded token.

The `user` method, contains logic that might seem abstract, so let's go through it:  In the first line, the `||=` operator is used to assign `@user`. Its purpose  is to "assign if not `nil`". Basically, if the `User.find()` returns an empty set or `decoded_auth_token` returns false, `@user` will be `nil`. Moving to the second line,  `user` method will either return the user or an error. In Ruby, the last line of the function is implicitly returned, so the command ends up returning the user object.

### Implementing helper methods into the controllers
 All the logic for handling JWT tokens has been laid down. It is time to implement it in the controllers and put it to actual use. The two most essential things that have to be implemented is a way of logging in the user and having his/her token returned and a `current_user` method to 'persist' the user and check the token for every request.
 
 
 #### Logging in users
 
 First, let's start with the logging in of the user:
 
 ```ruby
 # app/controllers/authentication_controller.rb

class AuthenticationController < ApplicationController
  skip_before_action :authenticate_request

  def authenticate
    command = AuthenticateUser.call(params[:email], params[:password])

    if command.success?
      render json: { auth_token: command.result }
    else
      render json: { error: command.errors }, status: :unauthorized
    end
  end
end
 ```
 The `authenticate` action will take the JSON parameters for e-mail and password through the `params` hash and pass them to the `AuthenticateUser` command. If the command succeeds, it will send the JWT  token back to the user.
 
 Let's put a endpoint for the action:
 
 ```ruby
   #config/routes.rb
   post 'authenticate', to: 'authentication#authenticate'
 ```
 
 
#### Authorizing requests
 
 To put the token to use, there must be a `current_user` method that will 'persist' the user. In order to have `current_user` available to all controllers, it has to be declared in the `ApplicationController`:

```ruby
#app/controllers/application_controller.rb
class ApplicationController < ActionController::API
 before_action :authenticate_request
  attr_reader :current_user

  private

  def authenticate_request
    @current_user = AuthenticateApiRequest.call(request.headers).result
    render json: { error: 'Not Authorized' }, status: 401 unless @current_user
  end
end
```
By using `before_action`, the server calls the and passes the request headers (using the built-in `request.headers`) to `AuthenticateApiRequest` every time the user makes a request. The results are returned to the `@current_user`, making it available to all controllers inheriting from `ApplicationController`.


### Does it work?

 Let's test if everything is working properly. First, create a user.
 Start the rails console in the application's root directory:
```bash
rails c
```  
Insert a user to the console:
```bash
User.create!(email: 'example@mail.com' , password: '123123123' , password_confirmation: '123123123')
```  

Now, open Postman or any other tool for making requests to an API and post the credentials to `localhost:3000/authorize`. Here is how teh request should look:

![request](https://raw.githubusercontent.com/pluralsight/guides/master/images/71e8f847-fd33-420a-a8bc-6bba399589f7.PNG)

 And you will get your token returned:

![response](https://raw.githubusercontent.com/pluralsight/guides/master/images/c858cc01-f723-47aa-8284-fb71090a43fb.PNG)

Great! 
