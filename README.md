# Nenv

Using ENV in Ruby is like using raw SQL statements - it feels wrong, because it is.

If you agree, this gem is for you.

## The benefits over using ENV directly:

- much friendlier stubbing in tests
- you no longer have to care whether false is "0" or "false" or whatever
- NO MORE ALL CAPS EVERYWHERE!
- keys become methods
- namespaces which can be passed around as objects
- you can subclass!
- you can marshal/unmarshal your own types automatically!
- strict mode saves you from doing validation yourself
- and there's more to come...


## Installation

Add this line to your application's Gemfile:

```ruby
gem 'nenv'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install nenv


## Examples !!!


### Automatic booleans

You no longer have to care whether the value is "0" or "false" or "no" or "FALSE" or ... whatever

```ruby
t.verbose = (ENV['CI'] == 'true')
ok = ENV['RUBYGEMS_GEMDEPS'] == "1" || ENV.key?('BUNDLE_GEMFILE']
ENV['DEBUG'] = "true"

```

now becomes:

```ruby
t.verbose = Nenv.ci?
gemdeps = Nenv.rubygems_gemdeps? || Nenv.bundle_gemfile?
Nenv.debug = true
```

### "Namespaces"

```ruby
puts ENV['GIT_BROWSER`]
puts ENV['GIT_PAGER`]
puts ENV['GIT_EDITOR`]
```

now becomes:

```ruby
git = Nenv :git
puts git.browser
puts git.pager
puts git.editor
```

### Custom type handling

```ruby
paths = [ENV['GEM_HOME`]] + ENV['GEM_PATH'].split(':')
enable_logging if Integer(ENV['WEB_CONCURRENCY']) > 1
mydata = YAML.load(ENV['MY_DATA'])
ENV['VERBOSE'] = debug ? "1" : nil
```

can become:

```ruby
# setup
gem = Nenv :gem
gem.instance.create_method(:path) { |p| p.split(':') }

web = Nenv :web
web.instance.create_method(:concurrency) { |c| Integer(c) }

my = Nenv :my
my.instance.create_method(:data) { |d| YAML.load(d) }

Nenv.instance.create_method(:verbose=) { |v| v ? 1 : nil }

# and then you can simply do:

paths = [gem.home] + gem.path
enable_logging if web.concurrency > 1
mydata = my.data
Nenv.verbose = debug
```

### Automatic conversion to string

```ruby
ENV['RUBYGEMS_GEMDEPS'] = 1  # TypeError: no implicit conversion of Fixnum into String
```

No automatically uses `to_s`:

```ruby
Nenv.rubygems_gemdeps = 1  # no problem here
```


### Custom assignment

```ruby
data = YAML.load(ENV['MY_DATA'])
data[:foo] = :bar
ENV['MY_DATA'] = YAML.dump(data)
```

can now become:

```ruby
my = Nenv(:my)
my.instance.create_method(:data) { |d| YAML.load(d) }
my.instance.create_method(:data=) { |d| YAML.dump(d) }

data = my.data
data[:foo] = :bar
my.data = data
```

### Strict mode

```ruby
fail 'home not allowed' if ENV['HOME'] = Dir.pwd  # BUG! Assignment instead of comparing!
puts ENV['HOME'] # Now contains clobbered value
```

Now, clobbering can be prevented:

```ruby
env = Nenv::Environment.new
env.create_method(:home)

fail 'home not allowed' if env.home = Dir.pwd  # Fails with NoMethodError
puts env.home # works
```

### Mashup mode

You can first define all the load/dump logic globally in one place

```ruby
Nenv.instance.create_method(:web_concurrency) { |d| Integer(d) }
Nenv.instance.create_method(:web_concurrency=)
Nenv.instance.create_method(:path) { |p| Pathname(p.split(File::PATH_SEPARATOR)) }
Nenv.instance.create_method(:path=) { |array| array.map(&:to_s).join(File::PATH_SEPARATOR) }

# And now, anywhere in your app:

Nenv.web_concurrency += 3
Nenv.path += Pathname.pwd + "foo"

```

## NOTES

Still, avoid using environment variables if you can.

At least, avoid actually setting them - especially in multithreaded apps.

As for Nenv, while you can access the same variable with or without namespaces,
filters are tied to instances, e.g.:

```ruby
Nenv.instance.create_method(:foo_bar) { |d| Integer(d) }
Nenv('foo').instance.create_method(:bar) { |d| Float(d) }
env = Nenv::Environment.new(:foo).tap { |e| e.create_method(:bar) }
```

all work on the same variable, but each uses a different filter for reading the value.


## What's wrong with ENV?

Well sure, having ENV act like a Hash is much better than calling "getenv".

Unfortunately, the advantages of using ENV make no sense:

1) it's faster but ... environment variables are rarely used thousands of times in tight loops
2) it's already an object ... but there's not much you can do with it
3) it's globally available ... but you can't isolate it in tests (you need to reset it every time)
4) you can use it to set variables ... but it's named like a const
5) it allows you to use keys regardless of case ... but by convention lowercase shouldn't be used except for local variables (which are only really used by shell scripts)
6) it's supposed to look ugly to discourage use ... but often your app/gem is forced to use them anyway
7) it's a simple class ... but either you encapsulate it in your own classes - or all the value mapping/validation happens everywhere you want the data


But the BIGGEST disadvantage is in specs, e.g.:

```ruby
allow(ENV).to receive(:[]).with('MY_VARIABLE').and_return("old data")
allow(ENV).to receive(:[]=).with('MY_VARIABLE', "new data")
```

which could instead be completely isolated as:

```ruby
let(:env) { instance_double(Nenv::Environment) }
before { allow(Nenv::Environment).to receive(:new).with(:my).and_return(env) }

allow(env).to receive(:variable).and_return("old data")
allow(env).to receive(:variable=).with("new data")
```


## Contributing

1. Fork it ( https://github.com/[my-github-username]/nenv/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request