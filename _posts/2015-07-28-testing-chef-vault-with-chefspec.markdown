---
layout: post
title:  "Testing Chef Vault with ChefSpec"
date:   2015-07-16 16:00:00
categories: ruby chef-vault chefspec chef
---

ChefSpec is a great tool to help developers write RSpec tests for their cookbooks. 
This makes testing core Chef resources a breeze. However, when it comes to a tool
like Chef Vault it's not immediately clear how to stub the vault or the method
calls to make this happen. This post covers how to stub `chef_vault_item` 
helper method provided by the
[chef-vault](https://supermarket.chef.io/cookbooks/chef-vault) cookbook.

For those not familiar with `chef_vault_item` it is a streamlined way to load a
Chef Vault in a recipe. For example, to load a vault called 'secrets' with a
vault item named 'password' write `chef_vault_item('secrets', 'password')`.
The helper method takes care of validating the vault's existence, requires
the `chef-vault` gem, and will automatically fallback to a standard Chef
Data Bag if a vault is not found.

The following examples assume you have some experience with RSpec/ChefSpec.

Continuing from the example given above, let's assume the recipe looks like this:

```ruby
include_recipe 'chef-vault'

password = chef_vault_item('secrets', 'password')
...
```

If tests are run against this recipe you will encounter errors and it's probably
a bit misleading.

```ruby
Chef::Exceptions::InvalidDataBagPath
------------------------------------
Data bag path '/var/folders/f5/2skb8m_10h55jtxn4ymk27gdrmb69d/T/d20150728-13935-1l7o87h/data_bags' is invalid

Cookbook Trace:
---------------
  /var/folders/f5/2skb8m_10h55jtxn4ymk27gdrmb69d/T/d20150728-13935-1l7o87h/cookbooks/chef-vault/libraries/chef_vault_item.rb:33:in `chef_vault_item'
  /var/folders/f5/2skb8m_10h55jtxn4ymk27gdrmb69d/T/d20150728-13935-1l7o87h/cookbooks/my_cookbook/recipes/default.rb:18:in `from_file'

Relevant File Content:
----------------------
/var/folders/f5/2skb8m_10h55jtxn4ymk27gdrmb69d/T/d20150728-13935-1l7o87h/cookbooks/chef-vault/libraries/chef_vault_item.rb:

 26:    #   chef_vault_item("secrets", "dbpassword")
 27:    # Instead of:
 28:    #   ChefVault::Item.load("secrets", "dbpassword")
 29:    #
 30:    # Falls back to normal data bag item loading if the item isn't actually a
 31:    # vault item.
 32:    def chef_vault_item(bag, item)
 33>>     if Chef::DataBag.load(bag).key?("#{item}_keys")
 34:        # We have a vault item
 35:        begin
 36:          require 'chef-vault'
 37:        rescue LoadError
 38:          raise("Missing gem 'chef-vault', use recipe[chef-vault] to install it first.")
 39:        end
 40:        ChefVault::Item.load(bag, item)
 41:      else
 42:        # We don't have a vault item, it must be a regular data bag

```

The data bag path has nothing to do with the problem. Notice the line the arrows
point to (line 33). The `chef-vault` cookbook is trying to do some pre-checking 
to see if a data bag looks like a vault. The first step to testing the vault is
to stub this pre-check and make it believe the vault exists.

```ruby
require 'spec_helper'
# Requiring 'chef-vault' is important. If not here, in spec_helper is fine.
require 'chef-vault'

describe 'my_cookbook::default' do
  before do
    # Stub the pre-check. Load the vault name ('secrets') and return the
    # item name with _keys appended ('password_keys' in this case).
    allow(Chef::DataBag).to receive(:load).with('secrets')
      .and_return('password_keys' => {})
  end
```

Now that we have stubbed the pre-check we can stub the actual vault item. This 
part is definitely not obvious and I only learned what needed stubbed by looking
at chef-vault cookbook code on GitHub. We need to stub `ChefVault::Item.load` and
return some sample data. 

```ruby
require 'spec_helper'
require 'chef-vault'

describe 'my_cookbook::default' do
  before do
    allow(Chef::DataBag)
      .to receive(:load).with('secrets').and_return('password_keys' => {})
    allow(ChefVault::Item)
      .to receive(:load).with('secrets', 'password').and_return(secret_data)  
  end
  let(:secret_data) do
    {
      'id'       => 'password',
      'password' => 'my_super_secret'
    } 
  end
```

Now you can write spec tests as normal and won't have errors. If the recipe went
on to create a config file including the password from the vault, you could write
a `render_file` expectation:

```ruby
it { is_expected.to render_file('/path/to/config').with_content('my_super_secret') }
```

Final Tip: If you have a recipe with a single vault but multiple items you will 
need to alter the first `allow` statement slightly. Simply copy/pasting the 
statement will lead to unexpected results. For example, let's assuming you have 
the same 'secrets' vault with 'dbpassword' as a second item. 

```ruby
allow(Chef::DataBag)
  .to receive(:load).with('secrets').and_return('password_keys' => {}, 'dbpassword_keys' => {})
```

After this you will want to duplicate and alter the second `allow` statement to 
return the sample data for the second vault item.

* * *

Unfortunately I don't have comments enabled on my blog yet. If you have comments, please send them to
[@drewblessing](https://twitter.com/drewblessing) on Twitter.
