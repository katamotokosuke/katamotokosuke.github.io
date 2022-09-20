---
layout: post
title: DDDを3分で理解する
which_category: diary
---


## はじめに
DDD(domain-driven design)について少し学習したのでまとめる
ここで記載することは主に軽量DDDと呼ばれるもので本質的なDDDについては取り上げない。

## What's the DDD?
DDDとはドメイン駆動設計のことで、ドメインの知識を元にソフトウェアを設計していこうという考えである。
ドメインロジックをとにかく外に漏らさない、内部に閉じ込めることにフォーカスしている設計。

## 登場人物
軽量DDDには主に以下の概念が登場する
- 値オブジェクト(ValueObject)
- エンティティ(Entity)
- サービスドメイン(Service)
- リポジトリ(Repository)
- アプリケーションサービス/ユースケース(ServiceApplication/UseCase)

この他にもファクトリーや仕様といった概念もあるが、そこはまぁ必要になるタイミングでWEBの記事を参照してみると良いと思う。
これらはすべてOOPにおけるクラスで表現されることが多い。各クラスの概略図は以下のようなイメージ。

レイヤードアーキテクチャにおける各クラスの依存関係

<img src="https://i.imgur.com/OVdqVqe.jpg">


### ValueObject
値オブジェクトは各プログラミング言語に用意されているプリミティブな型のように扱うオブジェクトのことである。

例：メルカリの商品IDを値オブジェクトで表したケース

※プロダクションのコードは@JvmInlineしたvalue classを使ったほうが良いと思うがわかりやすく通常のクラス定義にしている
```kotlin
class ItemID {
    private val value: String

    constructor(value: String)  {
        // メルカリの商品IDはm始まりであることを保証する
        // mのあとは[0-9]であることも検証したほうが良いが書くのが面倒なので省略
        if (value.first().toString() != "m") {
            throw Exception("itemIDのフォーマットが不正です")
        }
        this.value = value
    }
    
    override fun equals(other: Any?): Boolean {
        if (other is ItemID) {
            return this.value == other.value
        }
        return false
    }
}
```

ここでポイントなのは２つ。１つ目は `==`の演算子で等しい値であることが評価できることである。プリミティブな値を `'aaa' == 'bbb'`みたいに評価できるのと同じように定義した値オブジェクトは評価可能である必要がある。

２つ目は値オブジェクトの取りうる明確な知識が記載されている点。これまで通りitemIDをプリミティブな値で表現すると `m1244`みたいな感じで表現するしかなく、その場合だとprefixにm以外の値も許容できてしまう。
プログラムのどこかでm始まりであることを保証することになるが、ロジックが至るところに分散してしまうリスクがある。
しかし値オブジェクトを利用するとコンストラクタで表現することができ、コードの再利用も可能になる。
もし`m`以外に`r`が頭文字に追加されたとしてもこの値オブジェクトを修正するだけで済む。

また、値オブジェクトの `value`は不変であることも保証しておいたほうが良い。
例えば以下のように修正できてしまう値オブジェクト極力作らないほうが良い。

```kotlin
val itemId = ItemID("m1234")
itemId.changeId("m12345)
```

プリミティブな値が↑の例みたいに変更できないように値オブジェクトも変更できないようにあるべきである。
適切なのは以下のようなイメージ

```kotlin
var itemId = ItemID("m1234")
itemId = ItemID("m12345")
```


### Entity
エンティティは属性ではなく同一性によって識別されるオブジェクトとなる。
ユーザーID、商品IDなど識別子によりオブジェクトを区別する。値オブジェクトとの差は以下
- 可変であること
- 同じ属性であっても区別される
- 同一性により区別される

- 言葉ではわかりにくいので以下にサンプルを提示する。

```kotlin
data class Item(
    val itemId: ItemID,
    val title: Title, // Titleは値オブジェクトと想定
    val description: Description　// Descriptionは値オブジェクトと想定

) {
}
```

↑はItemのエンティティオブジェクトである。メルカリの商品はタイトルや説明欄が変更可能である。 このエンティティに`changeTitle`や`changeDescription`といった関数を用意し、Itemのエンティティを書き換えていくことで表現する。

また、ユーザーAとユーザーBがたまたま同じタイトルの商品を作ることがあるがサービス仕様としてそれは許容されている。
このエンティティクラスも`title`に同一の値が入ったとしても別の商品として扱えるという点が値オブジェクトとは異なる。

商品が同一の商品であることを担保する識別子として`itemID`というプロパティがある。これにより同一性を担保することになる。

DDDを実践していなくてもこのあたりはデータベースのselect結果やAPIのレスポンスについてORMやAPIClientが提供してたりするので馴染みのあるオブジェクトだと思う。




### Service
ドメインサービスは↑で列挙した値オブジェクトとエンティティに知識を集約させると不自然な振る舞いを記述するオブジェクトである。

ここでも例をまず提示する。
```kotlin
class RegisterService() {
    fun isValidBrandId() {
        // ブランドIDが存在しているか、カテゴリと一定の関連性を持っているかなどを検証
    }
    
    fun isOverHashTagCount() {
        // 付与されているハッシュタグの数が仕様以下であることを検証
    }
}
```

↑のような仕様があるかはメルカリ商品にあるかは不明だが、あると仮定して話を進めると、ブランドIDの検証可能なオブジェクトは今までの解釈だと値オブジェクトとエンティティのいずれかになる。
しかし、BrandIdオブジェクトに`isValidBrandId`があるのもややおかしいし、Brandエンティティに`isValidBrandId`があるのも自分自身に自分が正しいか？と聞いているようなものでおかしな話である。
この不自然さを解決するために値オブジェクトやエンティティにふさわしくないドメイン知識をServiceクラスに集約させる。

Serviceは放おっておくとここにありとあらゆる知識が書かれることになるので要注意である。
まず値オブジェクトやエンティティに知識を集約すること検討し、それでも不自然であればServiceオブジェクトの利用を検討するくらいがちょうどよいと思われる。


ここまで値オブジェクトとエンティティ、サービスの３つについてその概要を書いた。
これらはドメインオブジェクトと呼ばれDDDにおけるドメインモデルをできるだけ投影したオブジェクトになっていることが理想である。
これらのオブジェクトに知識を集約することで（外に知識を漏らさない！）、継続的に速度を落とさず開発に専念することができる。（知識が集約されている = 変更が容易）

### Repository
リポジトリはデータの永続化や再構築を担うオブジェクトである。
Spring Bootとか触っているとでてくるRepositoryとほぼ同じであると思って良い。

```kotlin
interface UserRepository() {
    fun getDetail(userId: UserId): User
    fun getList(userIds: List<UserId>): List<User>
    fun register(user: User): User
    fun delete(user: User): Boolean
    
}
class UserRepositoryImpl: UserRepository() {
    fun getDetail(userId: UserId): User {
        val query = ```
        SELECT * FROM users WHERE id = ?;
        ```
        val result = query.execute(query, [userId])
        new User(result.id, result.title ...etc)
    }
    
    // ...do something
}
```

後述するServiceApplication/UseCase層やService層で利用する感じになる。



### ServiceApplication/UseCase
ServiceApplicationやUseCaseはドメインオブジェクト、リポジトリを利用してサービス仕様を満たすためのクラスである。

こちらでも例を提示する。
```
class RegisterServiceApplication(
    private val registerService: RegisterService
    private val itemRepository: ItemRepository // IF) {
    
    fun register(input: RegisterInput): UserData {
        if (!registerService.isValidBrandId(input.brandId)) {
            throw Exception("ブランドが不正")
        }
        
        if (registerService.isOverHashTagCount(input.hashtags)) {
            throw Exception("ハッシュタグ多すぎ")
        }
        
        val user = User(input.hashtags, input.brandId) // ファクトリーを用意しても良いかもしれない
        val registerd = itemRepository.register(user)
        
        return registerd.toUserData() // DTOオブジェクトに変換 
    }
}
```

ServiceApplication/UseCaseはドメインオブジェクトを利用して仕様を満たすことに専念していることがわかる。
ここの層にはドメイン知識はない。あったらドメインオブジェクトに集約すること強くおすすめする。

また、返却値としてエンティティではなくDTOクラスのインスタンスが返却されることも見逃してはいけない。エンティティを外部に公開すると（ここでいう外部はUI層を想定）、外部からUserエンティティの関数の呼び出しが可能になり、Userエンティティに変更を行えてしまう。
これだとエンティティがどこから変更されるかが不安定になり、結果としてロジックが外部に漏れる、同じロジックがプロジェクト内で点在するといった問題のトリガーになりかねない。
なのでDTOクラスを返却することでエンティティを外から操作不可能にしてしまうのが良い。


## まとめ
DDDについて本当に軽くざっくりと概観を記した。

より詳しく知りたい方は以下の本を参照することをおすすめする。

軽量DDDをこの記事より詳細について理解するにはこの本がおすすめ

<a href="https://www.amazon.co.jp/%E3%83%89%E3%83%A1%E3%82%A4%E3%83%B3%E9%A7%86%E5%8B%95%E8%A8%AD%E8%A8%88%E5%85%A5%E9%96%80-%E3%83%9C%E3%83%88%E3%83%A0%E3%82%A2%E3%83%83%E3%83%97%E3%81%A7%E3%82%8F%E3%81%8B%E3%82%8B-%E3%83%89%E3%83%A1%E3%82%A4%E3%83%B3%E9%A7%86%E5%8B%95%E8%A8%AD%E8%A8%88%E3%81%AE%E5%9F%BA%E6%9C%AC-%E6%88%90%E7%80%AC-%E5%85%81%E5%AE%A3/dp/479815072X?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=1ZO0PG8J7BC0G&keywords=DDD%E5%AE%9F%E8%B7%B5%E5%85%A5%E9%96%80&qid=1663684969&sprefix=ddd%E5%AE%9F%E8%B7%B5%E5%85%A5%E9%96%80%2Caps%2C201&sr=8-3&linkCode=li2&tag=katamotokos09-22&linkId=6d89de654d093ed238c46683db0ba30e&language=ja_JP&ref_=as_li_ss_il" target="_blank"><img border="0" src="//ws-fe.amazon-adsystem.com/widgets/q?_encoding=UTF8&ASIN=479815072X&Format=_SL160_&ID=AsinImage&MarketPlace=JP&ServiceVersion=20070822&WS=1&tag=katamotokos09-22&language=ja_JP" ></a><img src="https://ir-jp.amazon-adsystem.com/e/ir?t=katamotokos09-22&language=ja_JP&l=li2&o=9&a=479815072X" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />

DDDのもっと深い部分まで知るにはこの本がおすすめ

<a href="https://www.amazon.co.jp/%E5%AE%9F%E8%B7%B5%E3%83%89%E3%83%A1%E3%82%A4%E3%83%B3%E9%A7%86%E5%8B%95%E8%A8%AD%E8%A8%88-Object-Oriented-SELECTION-%E3%83%B4%E3%82%A1%E3%83%BC%E3%83%B3%E3%83%BB%E3%83%B4%E3%82%A1%E3%83%BC%E3%83%8E%E3%83%B3/dp/479813161X?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=1ZO0PG8J7BC0G&keywords=DDD%E5%AE%9F%E8%B7%B5%E5%85%A5%E9%96%80&qid=1663684969&sprefix=ddd%E5%AE%9F%E8%B7%B5%E5%85%A5%E9%96%80%2Caps%2C201&sr=8-2&linkCode=li2&tag=katamotokos09-22&linkId=2fe59f94ee9c649f5ee486996e4ac44a&language=ja_JP&ref_=as_li_ss_il" target="_blank"><img border="0" src="//ws-fe.amazon-adsystem.com/widgets/q?_encoding=UTF8&ASIN=479813161X&Format=_SL160_&ID=AsinImage&MarketPlace=JP&ServiceVersion=20070822&WS=1&tag=katamotokos09-22&language=ja_JP" ></a><img src="https://ir-jp.amazon-adsystem.com/e/ir?t=katamotokos09-22&language=ja_JP&l=li2&o=9&a=479813161X" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />
