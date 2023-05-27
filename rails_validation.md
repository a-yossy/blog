## railsのvalidatesについて調べてみた

railsでは下記のように書くことでバリデーションを行える([コード参照元](https://railsguides.jp/active_record_validations.html#%E3%83%90%E3%83%AA%E3%83%87%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AE%E6%A6%82%E8%A6%81
))
```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end
```

この`validates`メソッドがどのように実装されているか気になったので調べてみた

### どこで定義されているのか
#### [ApplicationRecord](https://github.com/rails/rails/blob/main/railties/lib/rails/generators/rails/app/templates/app/models/application_record.rb.tt)
`ActiveRecord::Base`を継承しているクラスであり、`rails new`で自動生成される。
```ruby
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class
end
```

#### [ActiveRecord::Base](https://github.com/rails/rails/blob/66099147482ea431febf20936cec903f197d24be/activerecord/lib/active_record/base.rb)

```ruby
module ActiveRecord
  …
  class Base
    extend ActiveModel::Naming

    extend ActiveSupport::Benchmarkable
    extend ActiveSupport::DescendantsTracker

    extend ConnectionHandling
    extend QueryCache::ClassMethods
    extend Querying
    extend Translation
    extend DynamicMatchers
    extend DelegatedType
    extend Explain
    extend Enum
    extend Delegation::DelegateCache
    extend Aggregations::ClassMethods

    include Core
    include Persistence
    include ReadonlyAttributes
    include ModelSchema
    include Inheritance
    include Scoping
    include Sanitization
    include AttributeAssignment
    include ActiveModel::Conversion
    include Integration
    # ActiveRecord::Validationsがincludeされている
    include Validations
    include CounterCache
    include Attributes
    include Locking::Optimistic
    …
end
```

#### [ActiveRecord::Validations](https://github.com/rails/rails/blob/66099147482ea431febf20936cec903f197d24be/activerecord/lib/active_record/validations.rb)
```ruby
module ActiveRecord
  …
  module Validations
    extend ActiveSupport::Concern
    include ActiveModel::Validations
    …
  end
end
```


#### [ActiveModel::Validations](https://github.com/rails/rails/blob/66099147482ea431febf20936cec903f197d24be/activemodel/lib/active_model/validations.rb)

```ruby
require "active_support/core_ext/array/extract_options"

module ActiveModel
  …
  module Validations
    …
  end
  …
end

# validatesメソッドが実装されているファイルを読み込んでいる
Dir[File.expand_path("validations/*.rb", __dir__)].each { |file| require file }
```

#### [ActiveModel::Validations::ClassMethods#validates](https://github.com/rails/rails/blob/66099147482ea431febf20936cec903f197d24be/activemodel/lib/active_model/validations/validates.rb#L106-L128)
下記に`validates`メソッドが定義されている。

```ruby
module ActiveModel
  module Validations
    module ClassMethods
      …
      def validates(*attributes)
        defaults = attributes.extract_options!.dup
        validations = defaults.slice!(*_validates_default_keys)

        raise ArgumentError, "You need to supply at least one attribute" if attributes.empty?
        raise ArgumentError, "You need to supply at least one validation" if validations.empty?

        defaults[:attributes] = attributes

        validations.each do |key, options|
          key = "#{key.to_s.camelize}Validator"

          begin
            validator = const_get(key)
          rescue NameError
            raise ArgumentError, "Unknown validator: '#{key}'"
          end

          next unless options

          validates_with(validator, defaults.merge(_parse_validates_options(options)))
        end
      end
      …
    end
  end
end
```

### ActiveModel::Validations::ClassMethods#validatesの中身を読む

```ruby
def validates(*attributes)
  defaults = attributes.extract_options!.dup
  validations = defaults.slice!(*_validates_default_keys)

  raise ArgumentError, "You need to supply at least one attribute" if attributes.empty?
  raise ArgumentError, "You need to supply at least one validation" if validations.empty?
```

railsはアスタリスクを付けることで引数を配列に指定できる。そのため、`attributes`は`[:name, {presence=> true}]`となる。
[`extract_options!`](https://github.com/rails/rails/blob/main/activesupport/lib/active_support/core_ext/array/extract_options.rb)は下記のようになっている。

```ruby
class Hash
  def extractable_options?
    instance_of?(Hash)
  end
end

class Array
  def extract_options!
    if last.is_a?(Hash) && last.extractable_options?
      pop
    else
      {}
    end
  end
end
```
