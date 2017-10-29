# Light　Service

---

## light_serviceとは
[Railsコードを改善する7つの素敵なGem（翻訳）](https://techracho.bpsinc.jp/hachi8833/2017_10_25/47160)
で紹介されていた[interactor](https://github.com/collectiveidea/interactor)というgemの基本になったgem
のことである。

---

## interactor

interactorをrailsのAPplicationControllerで使用すると
以下のようなユーザー認証処理を...

```ruby
class SessionsController < ApplicationController
  def create
    if user = User.authenticate(session_params[:email], session_params[:password])
      session[:user_token] = user.secret_token
      redirect_to user
    else
      flash.now[:message] = "Please try again."
      render :new
    end
  end

  private

  def session_params
    params.require(:session).permit(:email, :password)
  end
end
```
@[3](認証ユーザーが取得できたら)
@[4](トークンをセッションに保存)

+++

こんなふうに見通しのいいように書き換えることができる。
どのようなアーキテクチャーなのだろうかを説明したい。


```ruby
class SessionsController < ApplicationController
  def create
    result = AuthenticateUser.call(session_params)

    if result.success?
      session[:user_token] = result.token
      redirect_to result.user
    else
      flash.now[:message] = t(result.message)
      render :new
    end
  end

  private

  def session_params
    params.require(:session).permit(:email, :password)
  end
end
```
@[3](この部分でユーザー認証処理が行われる)

---

## light　service

`interactor` には考え方まで丁寧に書かれていないが、その前進となった
<span style="font-family: Helvetica Neue; font-weight: bold; color:#ffffff"><span color:##e49436">light</span>service</span>には書かれていたのでそちらを読むことにしました。

+++

>Thank You

>**A very special thank you to Attila Domokos for his fantastic work on LightService. Interactor is inspired heavily by the concepts put to code by Attila.**
Interactor was born from a desire for a slightly simplified interface. We understand that this is a matter of personal preference, so please take a look at LightService as well!

---

# Light　Service

githubのREADMEに書いてある事をかいつまんで話します。

---

以下のように

<br>

- 注文合計に基づいて税率を表示 |
- 注文税を計算する |
- 税金の合計額が$ 200を超える場合は送料が無料になる |

という要件をみたす`TaxController#update`メソッドがApplicationControllerに
生えているとすると...

+++

大体こんな感じの実装になるかと思います。

```ruby
class TaxController < ApplicationController
  def update
    @order = Order.find(params[:id])
    tax_ranges = TaxRange.for_region(order.region)

    if tax_ranges.nil?
      render :action => :edit, :error => "The tax ranges were not found"
      return # Avoiding the double render error
    end

    tax_percentage = tax_ranges.for_total(@order.total)

    if tax_percentage.nil?
      render :action => :edit, :error => "The tax percentage  was not found"
      return # Avoiding the double render error
    end

    @order.tax = (@order.total * (tax_percentage/100)).round(2)

    if @order.total_with_tax > 200
      @order.provide_free_shipping!
    end

    redirect_to checkout_shipping_path(@order), :notice => "Tax was calculated successfully"
  end
end
```
@[4](regionによって税の範囲を決める)
@[11](合計額によって税率を求める)
@[18](税込みの合計金額を求める)
@[20-22]($200以上は送料無料にする)

---

SRP(単一責任の原則）を破っていて醜い。。。
まぁupdate処理の際に同時に実行しなきゃいけない状況なのでしょう....

---

そんな時に`light_service`を使うと以下の`CaluculatesTax.for_order`が全ての責任を追ってくれます。

+++

```ruby
class TaxController < ApplicationContoller
  def update
    @order = Order.find(params[:id])

    service_result = CalculatesTax.for_order(@order)

    if service_result.failure?
      render :action => :edit, :error => service_result.message
    else
      redirect_to checkout_shipping_path(@order), :notice => "Tax was calculated successfully"
    end

  end
end
```
@[5](おそらくですが、`for_order`クラス・メソッドは`call`クラス・メソッドの`alias`です。)

+++

`CalculatesTax`は以下のようになっています。
ServiceObjectようにcallクラス・メソッドを持っていますが、
そのcallメソッドの中にActionクラスを並べて順番に
実行出来るのが`light_service`の利点です。

+++

```ruby
class CalculatesTax
  extend LightService::Organizer

  def self.call(order)
    with(:order => order).reduce(
        LooksUpTaxPercentageAction,  # 注文合計に基づいて税率を表示
        CalculatesOrderTaxAction,    # 注文税を計算する
        ProvidesFreeShippingAction   # 税金の合計額が$ 200を超える場合は送料が無料になる
      )
  end
end

class LooksUpTaxPercentageAction
 extend LightService::Action
  # 注文合計に基づいて税率を表示
end

class CalculatesOrderTaxAction
 extend LightService::Action
　　 # 注文税を計算する
end

class ProvidesFreeShippingAction
 extend LightService::Action
  # 税金の合計額が$ 200を超える場合は送料が無料になる
end
```
@[6](注文合計にもとづいて税率を表示)
@[7](注文税を計算する)
@[8](税金の合計額が$200を超える場合は送料が無料になる)


---

## OrganizerとActionの関係
`extend LightService::Organizer`とあるように`CalclatesTax`クラスは`Organizer`クラスです。
`Organizer`が`Action`を一つづつ実行していくイメージです。

![OrganizerとActionの関係](https://raw.githubusercontent.com/adomokos/light-service/master/resources/organizer_and_actions.png)

---

## Context
まずContextを軽く紹介します。
`CaluculatesTax`は以下のような形をしていました。
ここで出て来る`with(:order => order)`は何をしているのでしょうか？
```ruby
class CalculatesTax
  extend LightService::Organizer

  def self.call(order)
    with(:order => order).reduce(
        LooksUpTaxPercentageAction,  # 注文合計に基づいて税率を表示
        CalculatesOrderTaxAction,    # 注文税を計算する
        ProvidesFreeShippingAction   # 税金の合計額が$ 200を超える場合は送料が無料になる
      )
  end
end
```

+++

with(:order => order)はContextを作成しています。
**Context**はHashクラスを継承したクラスです。

ソースコードを読むと`with(:order => order)`は`@context={:order => order}`
インスタンスを保持するContextクラスのインスタンスを返しています。

この**Context**はActionを語る上でも重要です。

---

## Organizer
Organizerモジュールをextendすることで２つのクラス・メソッドを使うことが出来るようになります。

|メソッド|内容|
|----------------|---------------|
|with(data = {} )|dataを保持したContextインスタンスを返す|
|reduce(*actions)|actionsを実行する|

この２つだけ
(細かく言えば、Macroでaliasesというのがあります)

+++

使い方は
```ruby
class CalculatesTax
  extend LightService::Organizer

  def self.call(order)
    with(:order => order).reduce(
        LooksUpTaxPercentageAction,  # 注文合計に基づいて税率を表示
        CalculatesOrderTaxAction,    # 注文税を計算する
        ProvidesFreeShippingAction   # 税金の合計額が$ 200を超える場合は送料が無料になる
      )
  end
end
```
@[2](LightService::Organizerをextendしてcallとwithクラス・メソッドを使えるようにする)
@[4-5,9-10](このブロックに実行したActionを順番に並べる。)
---

## Action
Actionクラスは以下のようになっています。
先程の最初のAction**注文合計にもとづいて税率を表示**パターンを見てみます。

ここでやっているのはregion毎で決まる税率の範囲(tax_ranges)が
`nil`で無いなら、注文の合計金額にもとづいて税率(tax_percentage)
を計算しています。

+++

```ruby
class LooksUpTaxPercentageAction
  extend LightService::Action
  expects :order
  promises :tax_percentage

  executed do |context|
    tax_ranges = TaxRange.for_region(context.order.region)
    context.tax_percentage = 0

    next context if object_is_nil?(tax_ranges, context, 'The tax ranges were not found')

    context.tax_percentage = tax_ranges.for_total(context.order.total)

    next context if object_is_nil?(context.tax_percentage, context, 'The tax percentage was not found')
  end

  def self.object_is_nil?(object, context, message)
    if object.nil?
      context.fail!(message)
      return true
    end

    false
  end
end
```
@[7](注文のregionに基づいて税金の範囲を求めて)
@[8](税率を初期化して)
@[12](合計額から税率を計算する)
@[10,14](仮にtax_rangesとtax_percentageがnilだったら失敗)
@[17-24](大事なのは処理が失敗したらcontext.fail!を呼ぶということ。呼び出し先でcontext.failure?で判断できるようにするためです)

+++
## expects
`context.order`
でorderを取得できる
<br>
context.orderはwith(:order=>order)で作られたのでした。

```ruby
class LooksUpTaxPercentageAction
  extend LightService::Action
  expects :order
  promises :tax_percentage

  executed do |context|
    # 処理
  end
end
```
@[3](内部的には@_expected_keys(配列)にconcatしています。)

+++
## promises
`context.tax_percentage = `
で値を与えることができるようになる。
<br>
もちろん`expects :tax_percentage`とするとその値を取得できるようになります。

```ruby
class LooksUpTaxPercentageAction
  extend LightService::Action
  expects :order
  promises :tax_percentage

  executed do |context|
    # 処理
  end
end
```
@[4](内部的には@_promised_keys(配列)にconcatしています。)

---

## Actionを実行時に途中で失敗したら

例えば、OrganizerがActionを４つ実行する場合で、最初の２つまでは成功したが、
３番目に失敗したとする。

![その後のActionは実行されない](https://raw.githubusercontent.com/adomokos/light-service/master/resources/fail_actions.png)

+++ 

```ruby
class LooksUpTaxPercentageAction
  extend LightService::Action
  expects :order
  promises :tax_percentage

  executed do |context|
    context.fail_and_return!("Failed to submit the order")
  end
end
```
@[7](これをよぶとcontext.failure?=trueとなり、context.message="Failed to submit the order"となる)

---

## Actionを条件によっては残りはスキップしたいことがある。

例えば、OrganizerがActionを4つ実行する場合で、最初の２つまで実行して
このり２つのActionは飛ばしたい時

![](https://raw.githubusercontent.com/adomokos/light-service/master/resources/skip_actions.png)

+++

```ruby
class LooksUpTaxPercentageAction
  extend LightService::Action
  expects :order
  promises :tax_percentage

  executed do |context|
    context.skip_remaining!("Everything is good, no need to execute the rest of the actions")
  end
end
```
@[7](context.failure?=falseであるのが重要。失敗はしていない。context.message="Everything is good, no need to execute the rest of the actions"となる。)

---

## Actionが失敗したらrollbackしたい
userエンティティの保存に失敗した。
Actionクラスの実行系のクラス・メソッドは`excutedとrolled_back`
だけである。

```ruby
class SaveEntities
  extend LightService::Action
  expects :user

  executed do |context|
    context.user.save!
  end

  rolled_back do |context|
    context.user.destroy
  end
end
```
@[9-11]

+++

`fail_with_rollback!`を使って以下のように書くこともできる

```ruby
class CallExternalApi
  extend LightService::Action

  executed do |context|
    api_call_result = SomeAPI.save_user(context.user)

    context.fail_with_rollback!("Error when calling external API") if api_call_result.failure?
  end
end
```
@[7]

+++

## 注意
context.failure?の値まではrollbackしない。
この値を変えることは失敗したという事実をなかったことにするのと同じである。

---

# Orchestrators

---

## Orchestrators
Organizerを束ねたいと思った時。
例えば、ELTワークフローを扱う時

>ETLとは
- Extract – 外部の情報源からデータを抽出 |
- Transform – 抽出したデータをビジネスでの必要に応じて変換・加工 |
- Load – 最終的ターゲット（すなわちデータウェアハウス）に変換・加工済みのデータをロード |

+++

# ETLの例
```ruby
class ExtractsTransformsLoadsData
  def self.run(connection)
    context = RetrievesConnectionInfo.call(connection)
    context = PullsDataFromRemoteApi.call(context)

    retrieved_items = context.retrieved_items
    if retrieved_items.empty?
      NotifiesEngineeringTeamAction.execute(context)
    end

    retrieved_items.each do |item|
      context[:item] = item
      TransformsData.call(context)
    end

    context = LoadsData.call(context)

    SendsNotifications.call(context)
  end
end
```
@[3](接続情報を取得する)
@[4](リモートAPIからデータを取得する (**Extract**))
@[6-9](なんでしょうね？笑)
@[11-14](データを変換する (**Transform**))
@[16](データをロードする (**Load**))
@[18](通知を送る)

+++

Orchestratorsを使うときれいになる

```ruby
class ExtractsTransformsLoadsData
  extend LightService::Orchestrator

  def self.run(connection)
    with(:connection => connection).reduce(steps)
  end

  def self.steps
    [
      RetrievesConnectionInfo,
      PullsDataFromRemoteApi,
      reduce_if(->(ctx) { ctx.retrieved_items.empty? }, [
        NotifiesEngineeringTeamAction
      ]),
      iterate(:retrieved_item, [
        TransformsData
      ]),
      LoadsData,
      SendsNotifications
    ]
  end
end
```
@[10](接続情報を取得する)
@[11](リモートAPIからデータを取得する (**Extract**))
@[12-14](なんでしょうね？笑)
@[15-17](データを変換する (**Transform**))
@[18](データをロードする (**Load**))
@[19](通知を送る)

---

## OrchestratorsとOrganizerとActionの関係
OrganizerとActionの関係と同じなので説明は省きます。

```
Orchestrators
   |-> run - steps
   Organizers
       |-> call - actions
       Actions
           |-> execute
```
@[1-2](Organizerwを実行していく)
@[3-4](Actionsを呼び出していく)
@[5-6](実際にActionを実行する)

---

# Logger
light_serviceのloggerは情報が豊富です。

+++

# 使い方
```ruby
LightService::Configuration.logger = Logger.new(STDOUT)
LightService::Configuration.logger = Logger.new('/dev/null')
```
@[1](標準出力したい場合)
@[2](結果を捨てる場合)

+++

# ログの例
```ruby
I, [DATE]  INFO -- : [LightService] - calling organizer <TestDoubles::MakesTeaAndCappuccino>
I, [DATE]  INFO -- : [LightService] -     keys in context: :tea, :milk, :coffee
I, [DATE]  INFO -- : [LightService] - executing <TestDoubles::MakesTeaWithMilkAction>
I, [DATE]  INFO -- : [LightService] -   expects: :tea, :milk
I, [DATE]  INFO -- : [LightService] -   promises: :milk_tea
I, [DATE]  INFO -- : [LightService] -     keys in context: :tea, :milk, :coffee, :milk_tea
I, [DATE]  INFO -- : [LightService] - executing <TestDoubles::MakesLatteAction>
I, [DATE]  INFO -- : [LightService] -   expects: :coffee, :milk
I, [DATE]  INFO -- : [LightService] -   promises: :latte
I, [DATE]  INFO -- : [LightService] -     keys in context: :tea, :milk, :coffee, :milk_tea, :latte
```
@[1](呼び出されたオーガナイザーが表示される)
@[2](Action実行前のコンテキストが表示される)
@[3](最初のActionが実行されたとわかる)
@[4](expectsのリストが表示される)
@[5](promisesのリストが表示される)
@[6](Action実行後のコンテキストが表示される)
@[7](次のActionが実行される)

+++

# 特徴

- Actionの実行順に表示される |
- Action実行前後のcontextが表示される |
- Actionが失敗したら詳細が表示される |
- Actionがスキップされたら詳細が表示される |

+++

# Actionの実行順に表示される
```ruby
I, [DATE]  INFO -- : [LightService] - calling organizer <TestDoubles::MakesTeaAndCappuccino>
I, [DATE]  INFO -- : [LightService] -     keys in context: :tea, :milk, :coffee
I, [DATE]  INFO -- : [LightService] - executing <TestDoubles::MakesTeaWithMilkAction>
I, [DATE]  INFO -- : [LightService] -   expects: :tea, :milk
I, [DATE]  INFO -- : [LightService] -   promises: :milk_tea
I, [DATE]  INFO -- : [LightService] -     keys in context: :tea, :milk, :coffee, :milk_tea
I, [DATE]  INFO -- : [LightService] - executing <TestDoubles::MakesLatteAction>
I, [DATE]  INFO -- : [LightService] -   expects: :coffee, :milk
I, [DATE]  INFO -- : [LightService] -   promises: :latte
I, [DATE]  INFO -- : [LightService] -     keys in context: :tea, :milk, :coffee, :milk_tea, :latte
```
@[3]
@[7]

+++

# Action実行前後のcontextが表示される
```ruby
I, [DATE]  INFO -- : [LightService] - calling organizer <TestDoubles::MakesTeaAndCappuccino>
I, [DATE]  INFO -- : [LightService] -     keys in context: :tea, :milk, :coffee
I, [DATE]  INFO -- : [LightService] - executing <TestDoubles::MakesTeaWithMilkAction>
I, [DATE]  INFO -- : [LightService] -   expects: :tea, :milk
I, [DATE]  INFO -- : [LightService] -   promises: :milk_tea
I, [DATE]  INFO -- : [LightService] -     keys in context: :tea, :milk, :coffee, :milk_tea
I, [DATE]  INFO -- : [LightService] - executing <TestDoubles::MakesLatteAction>
I, [DATE]  INFO -- : [LightService] -   expects: :coffee, :milk
I, [DATE]  INFO -- : [LightService] -   promises: :latte
I, [DATE]  INFO -- : [LightService] -     keys in context: :tea, :milk, :coffee, :milk_tea, :latte
```
@[2]
@[6]
@[10]

+++

# Actionが失敗したら詳細が表示される
```ruby
I, [DATE]  INFO -- : [LightService] - calling organizer <TestDoubles::MakesCappuccinoAddsTwoAndFails>
I, [DATE]  INFO -- : [LightService] -     keys in context: :milk, :coffee
W, [DATE]  WARN -- : [LightService] - :-((( <TestDoubles::MakesLatteAction> has failed...
W, [DATE]  WARN -- : [LightService] - context message: Can't make a latte from a milk that's too hot!
```
@[3]

+++

# Actionがスキップされたら詳細が表示される
```
I, [DATE]  INFO -- : [LightService] - calling organizer <TestDoubles::MakesCappuccinoSkipsAddsTwo>
I, [DATE]  INFO -- : [LightService] -     keys in context: :milk, :coffee
I, [DATE]  INFO -- : [LightService] - ;-) <TestDoubles::MakesLatteAction> has decided to skip the rest of the actions
I, [DATE]  INFO -- : [LightService] - context message: Can't make a latte with a fatty milk like that!
```
@[3]

---

# 残りの機能をざっくりと...

- context.messageにi18nが使える |
- ContextFactoryでOrganizerのmockが作れる |
- OrchestratorsにはOrganizerを並べる以外に６つの構文が用意されてる |

---

# まとめ

- interactorのベースになったらlight_serviceに関して説明しました |
- SRPを破っているメソッドはOrganizerとActionを使って解決すればいいと説明しました |
- OrganizerとActionの関係ならびに使い方を説明しました |
- Organizerを束ねるOrchestratorsの説明をしました |
- Loggerに関しての説明をしました |
- 残りの機能に関してざっくり説明しました |

---

# ご清聴ありがとうございました。
