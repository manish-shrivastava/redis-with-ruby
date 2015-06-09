Some Basic Commands
---

| Command | Overview |
| --- | --- |
| [sadd](http://redis.io/commands/sadd) | Add member to the set stored at key. If member is already a member of this set, no operation is performed. If key does not exist, a new set is created with member as its sole member. |
| [srem](http://redis.io/commands/srem) | Remove member from the set stored at key. If member is not a member of this set, no operation is performed. |
| [smembers](http://redis.io/commands/smembers) | Returns all the members of the set value stored at key. |
| [sinter](http://redis.io/commands/sinter) | Returns the members of the set resulting from the intersection of all the given sets. |
| [scard](http://redis.io/commands/scard) | Returns the set cardinality (number of elements) of the set stored at key. |
| [sismember](http://redis.io/commands/sismember) | Returns if member is a member of the set stored at key. |
| [multi](http://redis.io/commands/multi) | Marks the start of a transaction block. Subsequent commands will be queued for atomic execution using EXEC. |

Other commands
---

| Command | Overview |
| --- | --- |
| [zadd](http://redis.io/commands/zadd) | Adds the member with the specified score to the sorted set stored at key. If member is already a member of the sorted set, the score is updated and the element reinserted at the right position to ensure the correct ordering. If key does not exist, a new sorted set with the specified member as sole member is created. |
| [zrevrank](http://redis.io/commands/zrevrank) | Returns the rank of member in the sorted set stored at key, with the scores ordered from high to low. The rank (or index) is 0-based, which means that the member with the highest score has rank 0. |
| [zrevrange](http://redis.io/commands/zrevrange) | Returns the specified range of elements in the sorted set stored at key. The elements are considered to be ordered from the highest to the lowest score. Descending lexicographical order is used for elements with equal score. |
| [zscore](http://redis.io/commands/zscore) | Returns the score of member in the sorted set at key. |


Rails Uses
---
Account model using `ActiveRecord` and `Redis`. We'll be using sets to store the colleagues data.

```ruby

class Account < ActiveRecord::Base
  # subscribe a account
  def subscribe!(account)
    $redis.multi do
      $redis.sadd(self.redis_key(:subscribeing), account.id)
      $redis.sadd(account.redis_key(:subscribers), self.id)
    end
  end
  
  # unsubscribe a account
  def unsubscribe!(account)
    $redis.multi do
      $redis.srem(self.redis_key(:subscribeing), account.id)
      $redis.srem(account.redis_key(:subscribers), self.id)
    end
  end
  
  # accounts that self subscribes
  def subscribers
    account_ids = $redis.smembers(self.redis_key(:subscribers))
    User.where(:id => account_ids)
  end

  # accounts that subscribe self
  def subscribeing
    account_ids = $redis.smembers(self.redis_key(:subscribeing))
    User.where(:id => account_ids)
  end

  # accounts who subscribe and are being subscribed by self
  def colleagues
    account_ids = $redis.sinter(self.redis_key(:subscribeing), self.redis_key(:subscribers))
    User.where(:id => account_ids)
  end

  # does the account subscribe self
  def subscribed_by?(account)
    $redis.sismember(self.redis_key(:subscribers), account.id)
  end
  
  # does self subscribe account
  def subscribeing?(account)
    $redis.sismember(self.redis_key(:subscribeing), account.id)
  end

  # number of subscribers
  def subscribers_count
    $redis.scard(self.redis_key(:subscribers))
  end

  # number of accounts being subscribed
  def subscribeing_count
    $redis.scard(self.redis_key(:subscribeing))
  end
  
  # helper method to generate redis keys
  def redis_key(str)
    "account:#{self.id}:#{str}"
  end
end
```

and to sample uses like:

```
> %w[Manish Nishant].each{|name| Account.create(:name => name)}
=> ['Manish', 'Nishant']
> a, b = Account.all
=> [#<Account id: 1, name: "Manish">, #<Account id: 2, name: "Nishant">] 
> a.subscribe!(b)
=> [1, 1] 
> a.subscribing?(b)
=> true 
> b.subscribeed_by?(a)
=> true 
> a.subscribing
=> [#<Account id: 2, name: "Nishant">] 
> b.subscribers
=> [#<Account id: 1, name: "Manish">]
> a.colleagues
=> [] 
> b.subscribe!(a)
=> [1, 1] 
> a.colleagues
=> [#<Account id: 2, name: "Nishant">] 
> b.colleagues
=> [#<Account id: 1, name: "Manish">] 


```
