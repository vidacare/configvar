[![Build Status](https://travis-ci.org/heroku/configvar.svg?branch=master)](https://travis-ci.org/heroku/configvar)

# ConfigVar

Manage configuration values loaded from the environment.

## Usage

You need to define the configuration variables that `ConfigVar` loads from the
environment.

### Required variables

Required variables can be defined for string, integer and boolean values.
When the configuration is initialized from the environment a
`ConfigVar::ConfigError` exception is raised if a required variable isn't
defined.

```ruby
config = ConfigVar.define do
  required_string  :database_url
  required_int     :port
  required_bool    :enabled
end
```

With these definitions the environment must contain valid `DATABASE_URL`,
`PORT` and `ENABLED` variables.  If any value is invalid an `ArgumentError`
exception is raised.  Boolean values are case insensitive and may be `0` or
`false` for false and `1` or `true` for true.

### Optional variables

Optional variables can also be defined for string, integer and boolean values.
A default for each value must be provided.  The values must be of the same
type as the optional value or `nil` if a default value isn't needed.

```ruby
config = ConfigVar.define do
  optional_string  :name,      'Bob'
  optional_int     :age,       42
  optional_bool    :friendly,  true
end
```

Default values will be used if `NAME`, `AGE` or `FRIENDLY` environment
variables are not present when the configuration is loaded.

### Custom variables

When simple string, integer and boolean values aren't sufficient you can
provide a block to process values from the environment according to your
needs.  For example, if you want to load a required integer that must always
be 0 or greater you can provide a custom block to do the required validation.
It must take the environment to load values from and return a value to include
in the loaded configuration.

```ruby
config = ConfigVar.define do
  required_custom :age do |env|
    if value = env['AGE']
      value = Integer(value)
      if value < 0
        raise ArgumentError.new("#{value} for AGE must be a 0 or greater")
      end
      value
    else
      raise ConfigVar::ConfigError.new(name)
    end
  end
end
```

Custom functions can return multiple key/value pairs in a `Hash` and all of
them will be included in the loaded configuration.  For example, if you want
to extract parts of a URL and make them available as individual configuration
variables you can provide a custom block to do so:

```ruby
config = ConfigVar.define do
  optional_custom :url do |env|
    if value = env['URL']
      url = URI.parse(value)
      {host: url.host, port: url.port, path: url.path}
    else
      {host: 'localhost', port: nil, path: '/example'}
    end
  end
end
```

## Accessing configuration variables

The configuration object returned by `ConfigVar.define` exposes a basic
collection interface for accessing variables loaded from the environment,
based on the definitions provided.

```ruby
config = ConfigVar.define do
  required_string  :database_url
  required_int     :port
  required_bool    :enabled
end

database_url = config.database_url
port = config.port
enabled = config.enabled
```

A `NoMethodError` exception is raised if an unknown configuration variable is
accessed.

## Reloading configuration variables

The configuration object can be reloaded at any time from a `Hash` containing
environment variables.  For example, it's easy to load values from a file to
refresh an already initialized configuration object.  `ConfigVar.define` will
always initialize the `config` object from the environment.

```ruby
config = ConfigVar.define do
  required_string  :database_url
  required_int     :port
  required_bool    :enabled
end

env = load_values_from_file('my.config')
config.reload(env)
```
