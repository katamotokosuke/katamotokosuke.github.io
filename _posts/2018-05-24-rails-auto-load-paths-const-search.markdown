---
layout: post
title: Railsのautoload_pathsでの定数探索
which_category: tech
---

# 環境
- Ruby 2.5.1
- Rails 5.2.0

# autoload_pathsの概要
例:

```ruby
class Hoge < SuperHoge; end
```

のようにモデルを定義したとします。Hogeは未定義の場合定数作成され、定義済みであればオープンクラスされるためautoload_pathsの出番なしですが、SuperHogeのほうが未定義だった場合`autoload_paths`を元に定数を探索しに行きます。
`autoload_paths`があるので

```ruby
require 'super_hoge'
```
といちいち書かなくてもよくなっています。

```ruby
puts ActiveSupport::Dependencies.autoload_paths
RAILS_ROOT/app/assets
RAILS_ROOT/app/channels
RAILS_ROOT/app/controllers
RAILS_ROOT/app/controllers/concerns
RAILS_ROOT/app/helpers
RAILS_ROOT/app/jobs
RAILS_ROOT/app/mailers
RAILS_ROOT/app/models
RAILS_ROOT/app/models/concerns
...etc
```
みたいにActiveSupport::Dependencies.autoload_pathsの結果が探索の対象になります。
詳しくはwebで(https://railsguides.jp/autoloading_and_reloading_constants.html)



# 実際のソースコードを見てみる
rubyにはさまざまなフックメソッドが提供されていてその一つに定数が見つからないときに`Module#const_missing`というものがあります。これを`ActiveSupport::Dependencies::ModuleConstMissing`ではオーバーライドしています。このmoduleはrubyの`Module`に`include`されているのでRailsを使う際に定数が見つからないとこのメソッドのconst_missingが呼ばれることになります。なのでこれを見ていくことにします。

https://github.com/rails/rails/blob/5-2-0/activesupport/lib/active_support/dependencies.rb#L191


```ruby
def const_missing(const_name)
  from_mod = anonymous? ? guess_for_anonymous(const_name) : self
  Dependencies.load_missing_constant(from_mod, const_name)
end
```
from_modはどの名前空間に属するかを特定しています。そして定数探索の旅が始まる！


# ActiveSupport::Dependencies#load_missing_constant
https://github.com/rails/rails/blob/master/activesupport/lib/active_support/dependencies.rb#L489

2018/05/23時点のソースをのせる。

```ruby
    def load_missing_constant(from_mod, const_name)
      unless qualified_const_defined?(from_mod.name) && Inflector.constantize(from_mod.name).equal?(from_mod)
        raise ArgumentError, "A copy of #{from_mod} has been removed from the module tree but is still active!"
      end

      qualified_name = qualified_name_for from_mod, const_name
      path_suffix = qualified_name.underscore

      file_path = search_for_file(path_suffix)

      if file_path
        expanded = File.expand_path(file_path)
        expanded.sub!(/\.rb\z/, "".freeze)

        if loading.include?(expanded)
          raise "Circular dependency detected while autoloading constant #{qualified_name}"
        else
          require_or_load(expanded, qualified_name)
          raise LoadError, "Unable to autoload constant #{qualified_name}, expected #{file_path} to define it" unless from_mod.const_defined?(const_name, false)
          return from_mod.const_get(const_name)
        end
      elsif mod = autoload_module!(from_mod, const_name, qualified_name, path_suffix)
        return mod
      elsif (parent = from_mod.parent) && parent != from_mod &&
            ! from_mod.parents.any? { |p| p.const_defined?(const_name, false) }
        begin
          return parent.const_missing(const_name)
        rescue NameError => e
          raise unless e.missing_name? qualified_name_for(parent, const_name)
        end
      end
```


上から適当に見ていくと`qualified_name_for`で`Hoge::Fuga::Bar`のような文字列取得して、`underscore`で`hoge/fuga/bar`のような文字列を生成しています。それを`search_for_file`(あとでみる)でファイルを探しています。
その絶対パスを取得して、`require_or_load`を呼びだして定数をrequireかロードし、その後そのmoduleがreturnされます。

もしsearch_for_fileの返り値がなければ`autoload_module`を呼び出す。
`autoload_module`は --> 予想されるパスサフィックスに一致するディレクトリを検索して、提供されたモジュール名を自動ロードしようとします。 見つかった場合、モジュールは作成され、+ const_name +という名前の定数に+ from_mod +の定数に代入されます。 ディレクトリが再ロード可能なベースパスからロードされていれば、アンロードされる定数セットに追加されます。

らしいです。何かしらが代入されて場合、そのmoduleを返します。あとは例外処理なので省略。


# `search_for_file`を読む
https://github.com/rails/rails/blob/5-2-0/activesupport/lib/active_support/dependencies.rb#L414

```ruby
def search_for_file(path_suffix)
  path_suffix = path_suffix.sub(/(\.rb)?$/, ".rb".freeze)
  autoload_paths.each do |root|
    path = File.join(root, path_suffix)
    return path if File.file? path
  end
  nil
end

```

やっと出現しました`autoload_paths`。
実装は単純で、`autoload_paths`をぶん回して`path_suffix`(探索対象のモジュールの名前空間に対応するファイルパス)と各要素を結合します。それがファイルならばその文字列を返します。


# まとめ
支離滅裂に書いてきたが、`ActiveSupport::Dependencies::ModuleConstMissing#const_missing`では未定義の定数をautoload_pathsをもとに読み込んでいます。
