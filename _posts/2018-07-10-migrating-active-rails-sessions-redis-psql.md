# Rails session stores

At work, we have a rails application which is using redis (AWS ElastiCache in PRODUCTION) for a few different things. General caching, sidekiq jobs, and we also use it for our session store.

Work on this application is ramping down and resources will be moved away from it shortly. We have had instances where AWS ElastiCache wasn't great for the session store (network timeouts, unavailability, and automatic maintenance windows). Because of the lowered resources on the project, instead of architecting a more robust solution backed by redis, we opted to move the session store to ActiveRecord::SessionStore.

There's not a lot to this, but I thought it would be useful to document what I ended up implementing.

Because we didn't want this to impact users when it goes live, I ended up writing some code to manage translating and migrating any active sessions from redis to ActiveRecord::SessionStore.

## Redis store

The redis session store we originally used was pretty straight forward. We used the `redis-rails` gem. Sessions went in to redis under a "session" namespace. They were _essentially_ just ruby hashes (not really but for arguments sakes this is accurate enough).

## ActiveRecord::SessionStore

ActiveRecord::SessionStore treats session data a bit differently, though, and hashes the data values. Because of the discrepancy between the two session stores, I had to find a way to translate the session data from redis to ActiveRecord::SessionStore.

Thankfully Rails and ActiveRecord::SessionStore have a couple of nice features to help with this.

# Piecing things together

First things first, I added the `active_record_session_store` gem to the `Gemfile`, replacing `redis-rails` and created a new database migration to create the session table.

## The database migration

The first pass at this is exactly what you'd expect. Create the table and add some indexes. We'll come back to this later.

```ruby
class AddSessionsTable < ActiveRecord::Migration[5.1]
  def change
    create_table :sessions do |t|
      t.string :session_id, :null => false
      t.text :data
      t.timestamps
    end

    add_index :sessions, :session_id, :unique => true
    add_index :sessions, :updated_at
  end
end
```

## The session migration service

This is the more interesting piece.

```ruby
class SessionMigrationService
  def initialize
    url = "redis://#{Rails.application.secrets.redis[:host]}:#{Rails.application.secrets.redis[:port]}"
    @redis = Redis.new(url: url)
    @cache = ActiveSupport::Cache.lookup_store(:redis_store)
  end

  def execute
    redis_session_data.each do |hash|
      ActiveRecord::SessionStore::Session.new(
        session_id: hash.fetch('session_id'),
        data: hash.reject { |key| key == 'session_id' }
      ).save!
  end

  private

  attr_reader :cache, :redis

  def redis_session_data
    keys.map do |key|
      cache
        .read(key)
        .merge('session_id' => key.gsub(/session:/, ''))
    end
  end

  def keys
    redis.keys('session:*')
  end
end
```

There's not much to it. We instantiate a redis client and a cache client. The cache client here will allow us to very easily pull our session data out of redis as a ruby hash.

We pull the session key names out of redis, fetch the data that pairs to those keys out of the cache, and return them as an array of hashes. We loop over the array and use the session data in each hash to create and persist new ActiveRecord::SessionStore::Session objects.

## The final bit

We need to execute the service inside of our database migration.

```ruby
class AddSessionsTable < ActiveRecord::Migration[5.1]
  def change
    create_table :sessions do |t|
      t.string :session_id, :null => false
      t.text :data
      t.timestamps
    end

    add_index :sessions, :session_id, :unique => true
    add_index :sessions, :updated_at

    SessionMigrationService.new.execute
  end
end
```

And there you have it. Not much to it.
