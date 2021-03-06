---
layout: post
title:  "Ruby on Rails: Setup multiple associations with the same model"
date:   2018-04-04 17:06:33 +0800
categories: ruby rails
tags: [ ruby, rails ]
---
When I was just starting to learn Ruby on Rails, I had trouble setting up
multiple associations with the same model due to my lack of understanding and
reliance on Rail's "magical" generators. Here I'll show you how to implement it
and analyze how it happens. On this example there are two models, User and
TransferRequest. TransferRequest has attributes sender and receiver which are
instances of User.

User was generated by executing the command:

{% highlight shell %}
# Generates a User model.
rails g model User name
{% endhighlight %}

The User model will look like this:
{% highlight ruby %}
# app/models/transfer_request.rb
class User < ApplicationRecord
end
{% endhighlight %}

Lets generate the TransferRequest model by executing the command:
{% highlight shell %}
# Generates a TransferRequest model
rails g model TransferRequest sender:references receiver:references
{% endhighlight %}

The code for TransferRequest should look like this:
{% highlight ruby %}
# app/models/transfer_request.rb
class TransferRequest < ApplicationRecord
  belongs_to :sender
  belongs_to :receiver
end
{% endhighlight %}

A migration file to create the table for TransferRequest will also be generated.

{% highlight ruby %}
# db/migrations/20180404063005_create_transfer_requests.rb
class CreateTransferRequests < ActiveRecord::Migration[5.1]
  def change
    create_table :transfer_requests do |t|
      t.references :sender, foreign_key: true
      t.references :receiver, foreign_key: true

      t.timestamps
    end
  end
end
{% endhighlight %}

Lets run a migration to apply the changes to our database.

{% highlight shell %}
rails db:migrate # rake db:migrate if using Rails 4 and below.
{% endhighlight %}

When I was new to Ruby on Rails, I thought this would be enough to setup the
model TransferAsset and its association with **sender** and **receiver**.

Lets write a unit test that checks the **belongs_to** association but before
that lets first install the
[shoulda-matchers](https://github.com/thoughtbot/shoulda-matchers) gem. Add
these to your Gemfile:

{% highlight ruby %}
group :test do
  gem 'shoulda', '~> 3.5'
  gem 'shoulda-matchers', '~> 2.0'
end
{% endhighlight %}

Lets now write our unit test for the TransferRequest model. It should look like
the code below:

{% highlight ruby %}
# test/models/transfer_request_test.rb
require 'test_helper'

class TransferRequestTest < ActiveSupport::TestCase
  should belong_to :sender
  should belong_to :receiver
end
{% endhighlight %}

Now the run the test to check if the association was set up properly. Run the
command below.

{% highlight shell %}
rails test
{% endhighlight %}

The test will fail.

Lets analyze how this happened by first looking at the CreateTransferRequest
migration file and the changes it contributed to the **db/schema.rb** file after
running the migration. Running the migration modifies the **db/schema.rb**. This
piece of code will be added to it. Take note of the last two lines on the code
below.

{% highlight ruby %}
# db/schema.rb
create_table "transfer_requests", force: :cascade do |t|
  t.bigint "sender_id"
  t.bigint "receiver_id"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
  t.index ["receiver_id"], name: "index_transfer_requests_on_receiver_id"
  t.index ["sender_id"], name: "index_transfer_requests_on_sender_id"
end

add_foreign_key "transfer_requests", "sender"
add_foreign_key "transfer_requests", "receiver"
{% endhighlight %}

This instructs Rails Active Record to create the table **tansfer_request** with
columns **sender_id** and **receiver_id** which is represented by
TransferRequest as a model. The last two lines call the method
**add_foreign_key** which takes a table where the foreign keys reside as the
first argument and another table that will be references by the foreign key as
the second argument. Because of this Rails assumes that a table called
**sender** and **receiver** actually exists which
isn't true.

To fix this, we need to remove the last two lines on our code snippet to prevent
Rails from referencing the tables that don't exist however we can't just remove
the last two lines on the **db/schema.rb** because it will be inconsistent with
our migration file. We need to modify the CreateTransferRequest migration but
before that we must undo some changes to the database by rolling it back to the
state where the **transfer_request** table doesn't exist. Execute the command
below to revert the last migration and undo the changes to the databse.

{% highlight ruby %}
rails db:migrate # rake db:migrate if using Rails 4 and below.
{% endhighlight %}

This would revert the database and the **db/schema.rb** file to its previous
state. On the CreateTransferRequest migration file the value of the
**foreign_key** should be false or we can just ommit the part where the
**foreign_key** is passed since it is false by default like the code below.

{% highlight ruby %}
# db/migrations/20180404063005_create_transfer_requests.rb
class CreateTransferRequests < ActiveRecord::Migration[5.1]
  def change
    create_table :transfer_requests do |t|
      t.references :sender
      t.references :receiver

      t.timestamps
    end
  end
end
{% endhighlight %}

Now lets run the migration again.
{% highlight shell %}
rails db:migrate # Same as rake db:migrate
{% endhighlight %}

The **db/schema.rb** should be appended with the code below. It is expected not to
call the **add_foreign_key** method.

{% highlight ruby %}
# db/schema.rb
create_table "transfer_requests", force: :cascade do |t|
  t.bigint "sender_id"
  t.bigint "receiver_id"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
  t.index ["receiver_id"], name: "index_transfer_requests_on_receiver_id"
  t.index ["sender_id"], name: "index_transfer_requests_on_sender_id"
end
{% endhighlight %}

TransferRequest still doesn't know that its attributes **sender** and
**receiver** are instances of User so we pass **class_name: 'User'** as an
argument to the method **belongs_to** at the TransferRequest model.

{% highlight ruby %}
# app/models/transfer_request.rb
class TransferRequest < ApplicationRecord
  belongs_to :sender, class_name: 'User'
  belongs_to :receiver, class_name: 'User'
end
{% endhighlight %}


User doesn't know that it has many senders and receivers. Both from the table
**transfer_request**. Modify the User model to associate it with TransferRequest.
The User should look like this:

{% highlight ruby %}
# app/models/user.rb
class User < ApplicationRecord
  has_many :sender_transfer_request, class_name: 'TransferRequest',
    foreign_key: 'sender_id'
  has_many :receiver_transfer_request, class_name: 'TransferRequest',
    foreign_key: 'receiver_id'

  validates :email, presence: true, uniqueness: true
  validates :name, presence: true
end
{% endhighlight %}

Here the method **has_many** was called passing a symbol ending with
**_transfer_request** as the first argument following the naming convention for
Active Record association. The second argument tells Rails that it is an
instance of TranferRequest. The third argument specifies which the column of the
of the table **transfer_request** is the foreign key.

If you run the test now then it would succeed.
