# Rails session stores

At work, we have a rails application which is using redis (AWS ElastiCache) for a few different things. General caching, sidekiq jobs, and also as our session store.

Work on this application is ramping down and resources will be moved away from it shortly. We have had instances where AWS ElastiCache wasn't great for the session store (network timeouts, unavailability, and automatic maintenance windows). Because of the lowered resources on the project, instead of architecting a more robust solution backed by redis, we opted to move the session store to ActiveRecord::SessionStore.

There's not a lot to this, but I thought it was be useful to document what I ended up implementing.

Because we didn't want this to impact users when it goes live, I ended up writing some code to manage translating and migrating any active sessions from redis to ActiveRecord::SessionStore.

## Redis store

The redis session store was pretty straight forward. Keys went in to redis under a "session" namespace. They were _essentially_ just ruby hashes (not really but for arguments sakes this is accurate enough).

## ActiveRecord::SessionStore

ActiveRecord::SessionStore treats session data a bit differently, though, and hashes the data values. Because of the discrepancy between the two session stores, I had to find a way to translate the session data from redis to ActiveRecord::SessionStore.

Thankfully Rails has some very nice features to help as well as ActiveRecord::SessionStore.

# Piecing things together

First things first, I added the `active_record_session_store` gem to the `Gemfile` and created a new database migration to create the session table.

## The database migration

The first pass at this is exactly what you'd expect. Create the table, add some indexes. We'll come back to this later.

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
    @redis = Redis.new(url: "redis://#{Rails.application.secrets.redis[:host]}:#{Rails.application.secrets.redis[:port]}")
    @cache = ActiveSupport::Cache.lookup_store(:redis_store)
  end

  def execute
    keys.map do |key|
      @cache
        .read(key)
        .merge('session_id' => key.gsub(/session:/, ''))
    end
  end

  private

  def keys
    @keys ||= @redis.keys('session:*')
  end
end
```

There's not much to it. We instantiate a redis client and a cache client. The cache client here will allow us to very easily pull our session data out of redis as a ruby hash.

We pull any keys out of redis, fetch the data the pairs to those keys out of the cache, and return them as an array.

## The final migration

We need to use this session migration service inside of our database migration. We want to create the table, pull session data our of redis, convert it to an ActiveRecord::SessionStore session and then persist it. Thankfully, this is all painless.

```ruby
class AddSessionsTable < ActiveRecord::Migration[5.1]
  def up
    create_table :sessions do |t|
      t.string :session_id, :null => false
      t.text :data
      t.timestamps
    end

    add_index :sessions, :session_id, :unique => true
    add_index :sessions, :updated_at

    redis_session_data = SessionMigrationService.new.execute
    redis_session_data.each do |hash|
      ActiveRecord::SessionStore::Session.new(
        session_id: hash.fetch('session_id'),
        data:       hash.reject { |key| key == 'session_id' }
      ).save!
    end
  end

  def down
    raise ActiveRecord::IrreversibleMigration, 'ActiveRecord::SessionStore manages our sessions now!'
  end
end
```

And there you have it. Not much to it.
