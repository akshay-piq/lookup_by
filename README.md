# LookupBy

[![Gem Version](https://badge.fury.io/rb/lookup_by.png)][rubygems]
[![Build Status](https://secure.travis-ci.org/companygardener/lookup_by.png?branch=master)][travis]
[![Dependency Status](https://gemnasium.com/companygardener/lookup_by.png)][gemnasium]
[![Coverage Status](https://coveralls.io/repos/companygardener/lookup_by/badge.png?branch=master)][coveralls]
[![Code Climate](https://codeclimate.com/github/companygardener/lookup_by.png)][codeclimate]
[![Gittip](http://img.shields.io/gittip/companygardener.png)][gittip]

[rubygems]:    https://rubygems.org/gems/lookup_by
[travis]:      http://travis-ci.org/companygardener/lookup_by
[gemnasium]:   https://gemnasium.com/companygardener/lookup_by
[coveralls]:   https://coveralls.io/r/companygardener/lookup_by?branch=master
[codeclimate]: https://codeclimate.com/github/companygardener/lookup_by
[gittip]:      https://www.gittip.com/companygardener

# Overview

LookupBy is a thread-safe lookup table cache for ActiveRecord that reduces normalization pains and supports:

* configurable column names
* caching (read-through, write-through, least recently used (LRU))
* symbols

### Dependencies

* Rails 3.2+
* PostgreSQL

### Development

[github.com/companygardener/lookup_by][development]

### Source

git clone git://github.com/companygardener/lookup_by.git

### Bug reports

If you discover a problem with LookupBy, please let me know. However, I ask that you'd kindly review these guidelines before creating [Issues][]:

https://github.com/companygardener/lookup_by/wiki/Bug-Reports

If you find a security bug, please **_do not_** use the issue tracker. Instead, send an email to: thecompanygardener@gmail.com.

Please create [Issues][issues] to submit bug reports and feature requests.

_Provide a failing rspec test that concisely demonstrates the issue._

# Installation

Add this line to your application's Gemfile:

    gem "lookup_by"

And then execute:

    $ bundle

Or install it manually:

    $ gem install lookup_by


# Usage

### ActiveRecord Plugin

LookupBy adds two "macro" methods to `ActiveRecord::Base`

```ruby
lookup_by :column_name
# Defines .[], .lookup, and .is_a_lookup? class methods.

lookup_for :status
# Defines #status and #status= instance methods that transparently reference the lookup table.
# Defines .with_status(*names) and .without_status(*names) scopes on the model.
```

### Define the lookup model

```ruby
# db/migrate/201301010012_create_statuses_table.rb
create_table :statuses, primary_key: :status_id do |t|
  t.text :status, null: false
end

# Or use the shorthand
create_lookup_table :statuses

# UUID primary key
#   options[:id]    = :uuid
#
# SMALLSERIAL primary key
#   options[:small] = true
#
# Change the lookup column
#   options[:lookup_column] = "phone_number"
#   options[:lookup_type]   = :phone

# app/models/status.rb
class Status < ActiveRecord::Base
  lookup_by :status
end

# Aliases :name to the lookup attribute
Status.new(name: "paid")
```

### Define an association

```ruby
# db/migrate/201301010123_create_orders_table.rb
create_table :orders do |t|
  t.belongs_to :status
end

# app/models/order.rb
class Order < ActiveRecord::Base
  lookup_for :status
end
```

LookupBy creates methods that use the `status` attribute transparently:

```ruby
order = Order.new(status: "paid")

order.status
=> "paid"

order.status_id
=> 1

# Access the lookup object
order.raw_status
=> #<Status id: 1, status: "paid">

# Access the lookup value before type casting
order.status_before_type_cast
=> "paid"

# Look ma', no strings!
Order.column_names
=> ["order_id", "status_id"]
```

# Configuration

### Symbolize

Casts the attribute to a symbol. Enables the setter to take a symbol.

_Bad idea when the set of lookup values is large. Symbols are never garbage collected._

```ruby
class Order < ActiveRecord::Base
  lookup_for :status, symbolize: true
end

order = Order.new(status: "paid")

order.status
=> :paid

order.status = :shipped
=> :shipped
```

### Strict

By default, missing lookup values will raise an error.

```ruby
# Raise
#   Default
lookup_for :status

# this will raise a LookupBy::Error
Order.status = "non-existent status"

# Set to nil instead
lookup_for :status, strict: false
```

### Caching

The default is no caching. You can also cache all records or use an LRU.

_Note: caching is **per process**, make sure you think through the implications._

```ruby
# No caching - Not very useful
#   Default
lookup_by :column_name

# Cache all
#   Use for a small finite list (e.g. status codes, US states)
#
#   Defaults to no read-through
#     options[:find] = false
lookup_by :column_name, cache: true

# Cache N (with LRU eviction)
#   Use for large sets with uneven distribution (e.g. email domain, city)
#
#   Requires read-through
#     options[:find] = true
lookup_by :column_name, cache: 50
```

### Cache miss

You can enable read-throughs using the `:find` option.

```ruby
# Return nil
#   Default when caching all records
#
#   Skips the database for these methods:
#     .all, .count, .pluck
lookup_by :column_name, cache: true

# Find (read-through)
#   Required when caching N records
lookup_by :column_name, cache: 10
lookup_by :column_name, cache: true, find: true
```

### DB miss

You can enable write-throughs using the `:find_or_create` option.

_Note: This will only work if the primary key is a sequence and all columns but the lookup column are optional._

```ruby
# Return nil
#   Default
lookup_by :column_name

# Find or create
#   Useful for user-submitted fields that grow over time
#   e.g. user_agents, ip_addresses
lookup_by :column_name, cache: 20, find_or_create: true
```

### Raise on Miss

You can configure cache misses to raise a `LookupBy::RecordNotFound` error.

```ruby
# Return nil
#   Default
lookup_by :column_name, cache: true

# Raise if not found pre-loaded cache
lookup_by :column_name, cache: true, raise: true

# Raise if not found in DB, either
lookup_by :column_name, cache: true, find: true, raise: true
```


### Normalize values

```ruby
# Normalize
#   Run through the your attribute's setter
lookup_by :column_name, normalize: true
```

### Allow blank

Can be useful to handle `params` that are not required.

```ruby
# Allow blank
#   Treat "" different than nil
lookup_by :column_name, allow_blank: true
```

# Integration

### Cucumber

```ruby
# features/support/env.rb
require 'lookup_by/cucumber'
```

This provides: `Given I reload the cache for $plural_class_name`

### SimpleForm

```haml
= simple_form_for @order do |f|
  = f.input :status
  = f.input :status, :as => :radio_buttons
```

### Formtastic

```haml
= semantic_form_for @order do |f|
  = f.input :status
  = f.input :status, :as => :radio
```

## Testing

This plugin uses rspec and pry for testing. Make sure you have them installed:

    bundle

To run the test suite:

    rake app:db:test:prepare
    rake

# Contribute

  1. Fork
  2. Create a feature branch `git checkout -b new-hotness`
  3. Commit your changes `git commit -am 'Added some feature'`
  4. Push to the branch `git push origin new-hotness`
  5. Create a Pull Request

A list of authors can be found on the [Contributors][] page.

# License

Copyright © 2014 Erik Peterson

MIT License

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated
documentation files (the "Software"), to deal in the Software without restriction, including without limitation the
rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit
persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the
Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE
WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS
OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR
OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

[development]:  http://github.com/companygardener/lookup_by "LookupBy Development"
[issues]:       http://github.com/companygardener/lookup_by/issues "LookupBy Issues"
[license]:      http://github.com/companygardener/lookup_by/blob/master/MIT-LICENSE "LookupBy License"
[contributors]: http://github.com/companygardener/lookup_by/graphs/contributors "LookupBy Contributors"
