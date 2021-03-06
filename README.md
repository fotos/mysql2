# Mysql2 - A modern, simple and very fast Mysql library for Ruby - binding to libmysql

The Mysql2 gem is meant to serve the extremely common use-case of connecting, querying and iterating on results.
Some database libraries out there serve as direct 1:1 mappings of the already complex C API's available.
This one is not.

It also forces the use of UTF-8 [or binary] for the connection [and all strings in 1.9, unless Encoding.default_internal is set then it'll convert from UTF-8 to that encoding] and uses encoding-aware MySQL API calls where it can.

The API consists of two classes:

Mysql2::Client - your connection to the database

Mysql2::Result - returned from issuing a #query on the connection. It includes Enumerable.

## Installing

``` sh
gem install mysql2
```

You may have to specify --with-mysql-config=/some/random/path/bin/mysql_config

## Usage

Connect to a database:

``` ruby
# this takes a hash of options, almost all of which map directly
# to the familiar database.yml in rails
# See http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/MysqlAdapter.html
client = Mysql2::Client.new(:host => "localhost", :username => "root")
```

Then query it:

``` ruby
results = client.query("SELECT * FROM users WHERE group='githubbers'")
```

Need to escape something first?

``` ruby
escaped = client.escape("gi'thu\"bbe\0r's")
results = client.query("SELECT * FROM users WHERE group='#{escaped}'")
```

You can get a count of your results with `results.count`.

Finally, iterate over the results:

``` ruby
results.each do |row|
  # conveniently, row is a hash
  # the keys are the fields, as you'd expect
  # the values are pre-built ruby primitives mapped from their corresponding field types in MySQL
  # Here's an otter: http://farm1.static.flickr.com/130/398077070_b8795d0ef3_b.jpg
end
```

Or, you might just keep it simple:

``` ruby
client.query("SELECT * FROM users WHERE group='githubbers'").each do |row|
  # do something with row, it's ready to rock
end
```

How about with symbolized keys?

``` ruby
# NOTE: the :symbolize_keys and future options will likely move to the #query method soon
client.query("SELECT * FROM users WHERE group='githubbers'").each(:symbolize_keys => true) do |row|
  # do something with row, it's ready to rock
end
```

You can get the headers and the columns in the order that they were returned
by the query like this:

``` ruby
headers = results.fields # <= that's an array of field names, in order
results.each(:as => :array) do |row|
# Each row is an array, ordered the same as the query results
# An otter's den is called a "holt" or "couch"
end
```

## Cascading config

The default config hash is at:

``` ruby
Mysql2::Client.default_query_options
```

which defaults to:

``` ruby
{:async => false, :as => :hash, :symbolize_keys => false}
```

that can be used as so:

``` ruby
# these are the defaults all Mysql2::Client instances inherit
Mysql2::Client.default_query_options.merge!(:as => :array)
```

or

``` ruby
# this will change the defaults for all future results returned by the #query method _for this connection only_
c = Mysql2::Client.new
c.query_options.merge!(:symbolize_keys => true)
```

or

``` ruby
# this will set the options for the Mysql2::Result instance returned from the #query method
c = Mysql2::Client.new
c.query(sql, :symbolize_keys => true)
```

## Result types

### Array of Arrays

Pass the `:as => :array` option to any of the above methods of configuration

### Array of Hashes

The default result type is set to :hash, but you can override a previous setting to something else with :as => :hash

### Others...

I may add support for `:as => :csv` or even `:as => :json` to allow for *much* more efficient generation of those data types from result sets.
If you'd like to see either of these (or others), open an issue and start bugging me about it ;)

### Timezones

Mysql2 now supports two timezone options:

``` ruby
:database_timezone # this is the timezone Mysql2 will assume fields are already stored as, and will use this when creating the initial Time objects in ruby
:application_timezone # this is the timezone Mysql2 will convert to before finally handing back to the caller
```

In other words, if `:database_timezone` is set to `:utc` - Mysql2 will create the Time objects using `Time.utc(...)` from the raw value libmysql hands over initially.
Then, if `:application_timezone` is set to say - `:local` - Mysql2 will then convert the just-created UTC Time object to local time.

Both options only allow two values - `:local` or `:utc` - with the exception that `:application_timezone` can be [and defaults to] nil

### Casting "boolean" columns

You can now tell Mysql2 to cast `tinyint(1)` fields to boolean values in Ruby with the `:cast_booleans` option.

``` ruby
client = Mysql2::Client.new
result = client.query("SELECT * FROM table_with_boolean_field", :cast_booleans => true)
```

### Skipping casting

Mysql2 casting is fast, but not as fast as not casting data.  In rare cases where typecasting is not needed, it will be faster to disable it by providing :cast => false.

``` ruby
client = Mysql2::Client.new
result = client.query("SELECT * FROM table", :cast => false)
```

Here are the results from the `query_without_mysql_casting.rb` script in the benchmarks folder:

``` sh
                           user     system      total        real
Mysql2 (cast: true)    0.340000   0.000000   0.340000 (  0.405018)
Mysql2 (cast: false)   0.160000   0.010000   0.170000 (  0.209937)
Mysql                  0.080000   0.000000   0.080000 (  0.129355)
do_mysql               0.520000   0.010000   0.530000 (  0.574619)
```

Although Mysql2 performs reasonably well at retrieving uncasted data, it (currently) is not as fast as the Mysql gem.  In spite of this small disadvantage, Mysql2 still sports a friendlier interface and doesn't block the entire ruby process when querying.

### Async

NOTE: Not supported on Windows.

`Mysql2::Client` takes advantage of the MySQL C API's (undocumented) non-blocking function mysql_send_query for *all* queries.
But, in order to take full advantage of it in your Ruby code, you can do:

``` ruby
client.query("SELECT sleep(5)", :async => true)
```

Which will return nil immediately. At this point you'll probably want to use some socket monitoring mechanism
like EventMachine or even IO.select. Once the socket becomes readable, you can do:

``` ruby
# result will be a Mysql2::Result instance
result = client.async_result
```

NOTE: Because of the way MySQL's query API works, this method will block until the result is ready.
So if you really need things to stay async, it's best to just monitor the socket with something like EventMachine.
If you need multiple query concurrency take a look at using a connection pool.

### Row Caching

By default, Mysql2 will cache rows that have been created in Ruby (since this happens lazily).
This is especially helpful since it saves the cost of creating the row in Ruby if you were to iterate over the collection again.

If you only plan on using each row once, then it's much more efficient to disable this behavior by setting the `:cache_rows` option to false.
This would be helpful if you wanted to iterate over the results in a streaming manner. Meaning the GC would cleanup rows you don't need anymore as you're iterating over the result set.

### Streaming

`Mysql2::Client` can optionally only fetch rows from the server on demand by setting `:stream => true`. This is handy when handling very large result sets which might not fit in memory on the client.

``` ruby
result = client.query("SELECT * FROM really_big_Table", :stream => true)
```

There are a few things that need to be kept in mind while using streaming:

* `:cache_rows` is ignored currently. (if you want to use `:cache_rows` you probably don't want to be using `:stream`)
* You must fetch all rows in the result set of your query before you can make new queries. (i.e. with `Mysql2::Result#each`)

Read more about the consequences of using `mysql_use_result` (what streaming is implemented with) here: http://dev.mysql.com/doc/refman/5.0/en/mysql-use-result.html.

## ActiveRecord

To use the ActiveRecord driver (with or without rails), all you should need to do is have this gem installed and set the adapter in your database.yml to "mysql2".
That was easy right? :)

NOTE: as of 0.3.0, and ActiveRecord 3.1 - the ActiveRecord adapter has been pulled out of this gem and into ActiveRecord itself. If you need to use mysql2 with
Rails versions < 3.1 make sure and specify `gem "mysql2", "~> 0.2.7"` in your Gemfile

## Asynchronous ActiveRecord

Please see the [em-synchrony](https://github.com/igrigorik/em-synchrony) project for details about using EventMachine with mysql2 and Rails.

## Sequel

The Sequel adapter was pulled out into Sequel core (will be part of the next release) and can be used by specifying the "mysql2://" prefix to your connection specification.

## EventMachine

The mysql2 EventMachine deferrable api allows you to make async queries using EventMachine,
while specifying callbacks for success for failure. Here's a simple example:

``` ruby
require 'mysql2/em'

EM.run do
  client1 = Mysql2::EM::Client.new
  defer1 = client1.query "SELECT sleep(3) as first_query"
  defer1.callback do |result|
    puts "Result: #{result.to_a.inspect}"
  end

  client2 = Mysql2::EM::Client.new
  defer2 = client2.query "SELECT sleep(1) second_query"
  defer2.callback do |result|
    puts "Result: #{result.to_a.inspect}"
  end
end
```

## Lazy Everything

Well... almost ;)

Field name strings/symbols are shared across all the rows so only one object is ever created to represent the field name for an entire dataset.

Rows themselves are lazily created in ruby-land when an attempt to yield it is made via #each.
For example, if you were to yield 4 rows from a 100 row dataset, only 4 hashes will be created. The rest will sit and wait in C-land until you want them (or when the GC goes to cleanup your `Mysql2::Result` instance).
Now say you were to iterate over that same collection again, this time yielding 15 rows - the 4 previous rows that had already been turned into ruby hashes would be pulled from an internal cache, then 11 more would be created and stored in that cache.
Once the entire dataset has been converted into ruby objects, Mysql2::Result will free the Mysql C result object as it's no longer needed.

This caching behavior can be disabled by setting the :cache_rows option to false.

As for field values themselves, I'm workin on it - but expect that soon.

## Compatibility

The specs pass on my system (SL 10.6.3, x86_64) in these rubies:

* 1.8.7-p249
* ree-1.8.7-2010.01
* 1.9.1-p378
* ruby-trunk
* rbx-head - broken at the moment, working with the rbx team for a solution

The ActiveRecord driver should work on 2.3.5 and 3.0

## Yeah... but why?

Someone: Dude, the Mysql gem works fiiiiiine.

Me: It sure does, but it only hands you nil and strings for field values. Leaving you to convert
them into proper Ruby types in Ruby-land - which is slow as balls.


Someone: OK fine, but do_mysql can already give me back values with Ruby objects mapped to MySQL types.

Me: Yep, but it's API is considerably more complex *and* can be ~2x slower.

## Benchmarks

Performing a basic "SELECT * FROM" query on a table with 30k rows and fields of nearly every Ruby-representable data type,
then iterating over every row using an #each like method yielding a block:

These results are from the `query_with_mysql_casting.rb` script in the benchmarks folder

``` sh
 user       system     total       real
Mysql2
 0.750000   0.180000   0.930000 (  1.821655)
do_mysql
 1.650000   0.200000   1.850000 (  2.811357)
Mysql
 7.500000   0.210000   7.710000 (  8.065871)
```

## Development

To run the tests, you can use RVM and Bundler to create a pristine environment for mysql2 development/hacking.
Use 'bundle install' to install the necessary development and testing gems:

``` sh
bundle install
rake
```

The tests require the "test" database to exist, and expect to connect
both as root and the running user, both with a blank password:

``` sql
CREATE DATABASE test;
CREATE USER '<user>'@'localhost' IDENTIFIED BY '';
GRANT ALL PRIVILEGES ON test.* TO '<user>'@'localhost';
```

## Special Thanks

* Eric Wong - for the contribution (and the informative explanations) of some thread-safety, non-blocking I/O and cleanup patches. You rock dude
* Yury Korolev (http://github.com/yury) - for TONS of help testing the ActiveRecord adapter
* Aaron Patterson (http://github.com/tenderlove) - tons of contributions, suggestions and general badassness
* Mike Perham (http://github.com/mperham) - Async ActiveRecord adapter (uses Fibers and EventMachine)
