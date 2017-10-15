# Deploy a simple Hello Email app on Heroku using SendGrid's API

### Create a Heroku Account

#1

If you haven't already, then go ahead and Sign up for a free [Heroku](https://www.heroku.com/) account, and install the [Heroku toolbelt](https://devcenter.heroku.com/articles/heroku-cli) for your operating system.

#2

Once, you have a Heroku account, add your credit card info to your account. **Your card will not be charged for free plans** but this validation is needed in order to prevent spammers from abusing the free SendGrid plan.

#3

If you already have a Heroku account that you are signed into through your command line utility:

re-establish your credentials by: `$ heroku auth:logout`

...and then log back in: `$ heroku login`

If you are new to Heroku:

Open your command line utility and login: `$ heroku login`

### Create a Rails App

#4

The first thing you'll want to do is create a new Rails application by running the `rails new` command.

Lets, call the app, 'hello'

```ruby
$ rails new hello
```

### Create Heorku Application

#5

`cd` into the `hello` directory and run:

```
$ heroku create

Creating app... done, ⬢ morning-stream-10053
https://morning-stream-10053.herokuapp.com/ | https://git.heroku.com/morning-stream-10053.git
```

This creates a randomly generated application name and adds heroku as one of your remote destinations so you can push your repo to Heroku easily. If you want a custom name for your application, see Heroku's documentation [here](https://devcenter.heroku.com/articles/creating-apps#creating-a-named-app).

### Replace SQLite with PostgreSQL

#6

If you are using SQLite it must be replaced with PostgreSQL because Heroku does not support `sqlite3`. [Follow these directions](https://devcenter.heroku.com/articles/sqlite3).

### Test Deployment to Heroku

#7

If the setup has been done correctly, you can now try to deploy your application to Heroku

```
$ git push heroku master
```

### Install the SendGrid add-on from the command line:

#8

Lets add SendGrid to our app:

```
$ heroku addons:create sendgrid:starter

Creating sendgrid:starter on ⬢ morning-stream-10053... free
Use heroku addons:docs sendgrid to view documentation
```
#9

Verify that you installed SendGrid by typing the following on the command line:

```
$ heroku addons
```

### Configure Environment

#10

At the very end, add the following to your `config/environments/development.rb`:

```ruby
config.action_mailer.default_url_options = { host: 'localhost' }
```

#11

At the very end, add the following to your `config/environments/test.rb`:

```ruby
config.action_mailer.default_url_options = { host: 'localhost' }
```

#12

At the very end, add the following to your `config/environments/production.rb`

Replace "morning-stream-10053" with your app's name

```ruby
config.action_mailer.default_url_options = { host: 'morning-stream-10053.herokuapp.com' } #TODO
```

### SETUP ENVIRONMENT VARIABLES

#13

`Figaro` allows you to safely store and access sensitive credentials using variables.

In your `Gemfile` add `gem figaro` and run `$ bundle update`

Generate an `application.yml` file, which will be used to map environment variables to their values:

```
$ figaro install
```

Make sure that `config/application.yml` has been added to your `.gitignore` file,

#14

Open `config/application.yml` and add:

Note: Replace the sample values with yours, the **values are NOT added as STRINGS**

```ruby
#TODO:
SENDGRID_USERNAME: app02268941@heroku.com
SENDGRID_PASSWORD: 5qapohdfq1012
SENDGRID_API_KEY: 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
```

#15

Retrieve your SendGrid Username and Password and add them to `config/application.yml`:

```
$ heroku config:get SENDGRID_USERNAME
$ heroku config:get SENDGRID_PASSWORD
```

#16

In your app's dashboard in Heroku, on the upper left side you will see the SendGrid icon, click on the link, go to _Settings_, then _API Keys_ and generate an API key for your app. Add this to `config/application.yml`.

You will also need to add your API Key as a production variable to Heroku:

```
$ heroku config:add SENDGRID_API_KEY="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"
```

### SETUP MAILER

#17

Create a file in `config/initializers` named `setup_mail.rb`:

```
$ touch config/initializers/setup_mail.rb
```
add the following code:

```ruby
if Rails.env.development? || Rails.env.production?
  ActionMailer::Base.delivery_method = :smtp
  ActionMailer::Base.smtp_settings = {
    address:        'smtp.sendgrid.net',
    port:           '587',
    authentication: :plain,
    user_name:      ENV['SENDGRID_USERNAME'],
    password:       ENV['SENDGRID_PASSWORD'],
    domain:         'heroku.com',
    enable_starttls_auto: true
  }
end
```
The code in `config/initialize` runs when our app starts. We need to configure some special settings to send emails.

#18

Lets, generate a mailer

```ruby
$ rails generate mailer HelloMailer
```

#19

In your `Gemfile` add `gem sendgrid-ruby` and run `$ bundle update`

#20

In `app/mailers/hello_mailer.rb` add:

```ruby
require 'sendgrid-ruby'

class HelloMailer < ApplicationMailer
  include SendGrid

  def self.new_message(email)

    from = Email.new(email: 'replacethiswithyouremail@gmail.com') #TODO
    subject = 'Hello World from the SendGrid Ruby Library!'
    to = Email.new(email: email)
    content = Content.new(type: 'text/plain', value: 'Hello, Email!')
    mail = Mail.new(from, subject, to, content)

    sg = SendGrid::API.new(api_key: ENV['SENDGRID_API_KEY'])
    response = sg.client.mail._('send').post(request_body: mail.to_json)
    puts response.status_code
    puts response.body
    puts response.headers

  end
end
```

This is a simple example, but if you would like to to use a SendGrid Template, have a look at this [example](https://github.com/sendgrid/sendgrid-ruby/blob/master/USE_CASES.md)

### Testing

We can test the mailer in the Rails console:

```
$ rails console
2.3.3 :001 > HelloMailer.new_message('someone@gmail.com')
```

If everything is setup correctly, you should receive an email in your inbox within a couple of minutes.

### Using the email method:

You can use `HelloMailer.new_message()` in a controller or a model:

For example if you want to send a Hello Email when a _new user_ is created:


Call method from Controller:

```ruby
class UserController < ApplicationController
  def create
    @user = User.new(params[:user])

    if @user.save
      HelloMailer.new_message(@user.email)
    end
    # some code goes here....
  end
end
```

Or call method from Model:

```ruby
class User < ActiveRecord::Base
  after_create :send_hello_email

  private

  def send_hello_email
    HelloMailer.new_message(self.email)
  end
end
```

### Deploy to Heroku

```
$ git push heroku master
```

_Note: This example doesn't use a database, but if your application has a database then migrate your production database to Heroku_ - [instructions](https://devcenter.heroku.com/articles/getting-started-with-rails4#migrate-your-database)
