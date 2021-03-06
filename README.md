# ajax-datatables-rails

[![Build Status](https://travis-ci.org/antillas21/ajax-datatables-rails.svg?branch=master)](https://travis-ci.org/antillas21/ajax-datatables-rails)
[![Gem Version](https://badge.fury.io/rb/ajax-datatables-rails.svg)](http://badge.fury.io/rb/ajax-datatables-rails)

### Under new management

> Hi,
>
> New maintainer here. Just to let you know that we have
> released version 0.1.0 which includes breaking changes.
>
> Please check the [CHANGELOG](https://github.com/antillas21/ajax-datatables-rails/blob/master/CHANGELOG.md) to learn about these changes.
>
> All changes have been documented below.


Datatables is a nifty jquery plugin that adds the ability to paginate, sort, and search your html tables. When dealing with large tables (more than a couple hundred rows) however, we run into performance issues. These can be fixed by using server-side pagination, but this breaks some datatables functionality.

`ajax-datatables-rails` is a wrapper around datatable's ajax methods that allow synchronization with server-side pagination in a rails app. It was inspired by this [Railscast](http://railscasts.com/episodes/340-datatables). I needed to implement a similar solution in a couple projects I was working on so I extracted it out into a gem.

## ORM support

Currently `AjaxDatatablesRails` only supports `ActiveRecord` as ORM for performing database queries.

Adding support for `Sequel`, `Mongoid` and `MongoMapper` is a planned feature for this gem. If you'd be interested in contributing to speed development, please [open an issue](https://github.com/antillas21/ajax-datatables-rails/issues/new) and get in touch.

## Installation

Add these lines to your application's Gemfile:

    gem 'jquery-datatables-rails'
    gem 'rails-datatables'

And then execute:

    $ bundle

## Usage
*The following examples assume that we are setting up rails-datatables for an index of users from a `User` model*

### Generate
Run the following command:

    $ rails generate datatable User


This will generate a file named `user_datatable.rb` in `app/datatables`. Open the file and customize in the functions as directed by the comments.

Take a look [here](#generator-syntax) for an explanation about the generator syntax.

### Customize
```ruby
# uncomment the appropriate paginator module,
# depending on gems available in your project.
# include AjaxDatatablesRails::Extensions::Kaminari
# include AjaxDatatablesRails::Extensions::WillPaginate
# include AjaxDatatablesRails::Extensions::SimplePaginator

def sortable_columns
  # list columns inside the Array in string dot notation.
  # Example: 'users.email'
  @sortable_columns ||= []
end

def searchable_columns
  # list columns inside the Array in string dot notation.
  # Example: 'users.email'
  @searchable_columns ||= []
end
```

* For `paginator options`, just uncomment the paginator you would like to use, given
the gems bundled in your project. For example, if your models are using `Kaminari`, uncomment `AjaxDatatablesRails::Extensions::Kaminari`. You may remove all commented lines.
  * `SimplePaginator` is the most basic of them all, it falls back to passing `offset` and `limit` at the database level (through `ActiveRecord` of course, as that is the only ORM supported for the time being).

* For `sortable_columns`, assign an array of the database columns that correspond to the columns in our view table. For example `[users.f_name, users.l_name, users.bio]`. This array is used for sorting by various columns.

* For `searchable_columns`, assign an array of the database columns that you want searchable by datatables. For example `[users.f_name, users.l_name]`

This gives us:
```ruby
include AjaxDatatablesRails::Extensions::Kaminari

def sortable_columns
  @sortable_columns ||= ['users.f_name', 'users.l_name', 'users.bio']
end

def searchable_columns
  @searchable_columns ||= ['users.f_name', 'users.l_name']
end

```

### Map data
```ruby
def data
  records.map do |record|
    [
        # comma separated list of the values for each cell of a table row
        # example: record.attribute,
      ]
  end
end
```

This method builds a 2d array that is used by datatables to construct the html table. Insert the values you want on each column.

```ruby
def data
  records.map do |record|
    [
      record.f_name,
      record.l_name,
      record.bio
    ]
  end
end
```

#### Get Raw Records
```ruby
def get_raw_records
  # insert query here
end
```

This is where your query goes.

```ruby
def get_raw_records
  User.all
end
```

Obviously, you can construct your query as required for the use case the datatable is used. Example: `User.active.with_recent_messages`.

### Controller
Set up the controller to respond to JSON

```ruby
def index
  respond_to do |format|
    format.html
    format.json { render json: UserDatatable.new(view_context) }
  end
end
```

Don't forget to make sure the proper route has been added to `config/routes.rb`.

### View
* Set up an html `<table>` with a `<thead>` and `<tbody>`
* Add in your table headers if desired
* Don't add any rows to the body of the table, datatables does this automatically
* Add a data attribute to the `<table>` tag with the url of the JSON feed

The resulting view may look like this:

```erb
<table id="users-table", data-source="<%= users_path(format: :json) %>">
  <thead>
    <tr>
      <th>First Name</th>
      <th>Last Name</th>
      <th>Brief Bio</th>
    </tr>
  </thead>
  <tbody>
  </tbody>
</table>
```

### Javascript
Finally, the javascript to tie this all together. In the appropriate `js.coffee` file:

```coffeescript
$ ->
  $('#users-table').dataTable
    bProcessing: true
    bServerSide: true
    sAjaxSource: $('#users-table').data('source')
    sPaginationType: 'full_numbers'
    # optional, if you want full pagination controls.
    # Check dataTables documentation to learn more about
    # available options.
```

or, if you're using plain javascript:
```javascript
// users.js

jQuery(document).ready(function() {
  $('#users-table').dataTable({
    'bProcessing': true,
    'bServerSide': true,
    'sAjaxSource': $('#users-table').data('source'),
    'sPaginationType': 'full_numbers',
    // optional, if you want full pagination controls.
    // Check dataTables documentation to learn more about
    // available options.
  });
});
```

### Additional Notes

#### Options

An `AjaxDatatablesRails::Base` inherited class can accept an options hash at initialization. This provides room for flexibility when required. Example:

```ruby
class UnrespondedMessagesDatatable < AjaxDatatablesRails::Base
  # customized methods here
end

datatable = UnrespondedMessagesDatatable.new(view_context,
  { :foo => { :bar => Baz.new }, :from => 1.month.ago }
)

datatable.options
#=> { :foo => { :bar => #<Baz:0x007fe9cb4e0220> }, :from => 2014-04-16 19:55:28 -0700 }
```

#### Generator Syntax

Also, a class that inherits from `AjaxDatatablesRails::Base` is not tied to an existing model, module, constant or any type of class in your Rails app. You can pass a name to your datatable class like this:


```
$ rails generate datatable users
# returns a users_datatable.rb file with a UsersDatatable class

$ rails generate datatable contact_messages
# returns a contact_messages_datatable.rb file with a ContactMessagesDatatable class

$ rails generate datatable UnrespondedMessages
# returns an unresponded_messages_datatable.rb file with an UnrespondedMessagesDatatable class
```


In the end, it's up to the developer which model(s), scope(s), relationship(s) (or else) to employ inside the datatable class to retrieve records from the database.


## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Added some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
