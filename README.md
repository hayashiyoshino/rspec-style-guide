# RSpec スタイルガイド

## context と describe

describe と context は同じメソッドだが、次のように使い分けることで何をテストしているのかをわかりやすくできる。

- describe の引数にはテストの対象を書く
- context の引数にはテストを実行するための条件を書く

### 例

```ruby
describe Stack do
  let!(:stack) { Stack.new }

  describe '#push' do
    context '文字列をpushしたとき' do
      before { stack.push('value') }

      it '返り値がpushした値であること' do
        expect(stack).to eq 'value'
      end
    end

    context 'nilをpushした場合' do
      it 'ArgumentErrorになること' do
        expect { stack.push(nil) }.to raise_error(ArgumentError)
      end
    end
  end

  describe '#pop' do
    context 'スタックが空の場合' do
      it '返り値はnilであること' do        
        expect(stack.pop).to be_nil
      end
    end

    context 'スタックに値があるとき' do
      before do
        stack.push 'value1'
        stack.push 'value2'        
      end

      it '最後の値を取得すること' do
        expect(stack.pop).to eq 'value2'
      end
    end
  end
end
```

## FactoryGirl

FactoryGirlを利用した場合、各モデルのデフォルトのカラム値を設定することになる。このとき、各カラムの値がすべてランダム値となるように設定を書くとよい。その上で、必要な値のみをテスト中で明示的に指定することにより、「このテストで重要な値はなにか」がわかりやすくなる。

### よくない例

アカウントが有効化されているかどうかをactiveカラムで管理しているとする。このような、有効／無効を表すカラムが固定されているケースはよく見かける。

```ruby
FactoryGirl.define do
  factory :user do
    name 'willnet'
    active true
  end
end
```

```ruby
describe User, type: :model do
  describe '#send_message' do
    let!(:sender) { create :user, name: 'maeshima' }
    let!(:receiver) { create :user, name: 'kamiya' }

    it 'メッセージが正しく送られること' do
      expect { sender.send_message(receiver: receiver, body: 'hello!') }
        .to change { Message.count }.by(1)
    end
  end
end
```

このテストは`User#active`が`true`であることが暗黙的な条件になってしまっている。`sender.active #=> false`のときや`receiver.active #=> false`のときにどう振る舞うかを伝えることができていない。

さらに、このテストでは`name`を明示的に指定しているがこれは必要な指定なのだろうか？テストを読む人に余計なリソースを消費させてしまう、無駄なデータ指定はなるべく避けるのが好ましい。

### よい例

```ruby
FactoryGirl.define do
  factory :user do
    sequence(:name) { |i| "test#{i}"}
    active { [true, false].sample }
  end
end
```

```ruby
describe User, type: :model do
  describe '#send_message' do
    let!(:sender) { create :user }
    let!(:receiver) { create :user }

    it 'メッセージが正しく送られること' do
      expect { sender.send_message(receiver: receiver, body: 'hello!') }
        .to change { Message.count }.by(1)
    end
  end
end
```

このテストだと「`User#active`の戻り値が`User#send_message`の動作に影響しない」ということが(暗黙的にであるが)伝わる。もし`User#active`が影響するような修正が加えられた場合、CIで時々テストが失敗することに酔って、テストが壊れたことに気付けるはずだ。

## 日時を取り扱うテストを書く

日時を取り扱うテストを書く場合、絶対時間を使う必要がないケースであればなるべく現在日時からの相対時間を利用するのが良い。その方が実装の不具合に気づける可能性が増すからだ。

例として、先月に公開された投稿を取得するscopeと、そのテストを絶対時間を用いて記述する。

```ruby
class Post < ApplicationRecord
  scope :last_month_published, -> { where(publish_at: (Time.zone.now - 31.days).all_month) }
end
```

```ruby
require 'rails_helper'

RSpec.describe Post, type: :model do
  describe '.last_month_published' do
    let!(:april_1st) { create :post, publish_at: Time.zone.local(2017, 4, 1) }
    let!(:april_30th) { create :post, publish_at: Time.zone.local(2017, 4, 30) }

    before do
      create :post, publish_at: Time.zone.local(2017, 5, 1)
      create :post, publish_at: Time.zone.local(2017, 3, 31)
    end

    it 'return published posts in last month' do
      Timecop.travel(2017, 5, 6) do
        expect(Post.last_month_published).to contain_exactly(april_1st, april_30th)
      end
    end
  end
end
```

このテストは常に成功するが、実装にはバグが含まれている。

テストを相対日時に変更してみる。

```ruby
require 'rails_helper'

RSpec.describe Post, type: :model do
  describe '.last_month_published' do
    let!(:now) { Time.zone.now }
    let!(:last_beginning_of_month) { create :post, publish_at: 1.month.ago(now).beginning_of_month }
    let!(:last_end_of_month) { create :post, publish_at: 1.month.ago(now).end_of_month  }

    before do
      create :post, publish_at: now
      create :post, publish_at: 2.months.ago(now)
    end

    it 'return published posts in last month' do
      expect(Post.last_month_published).to contain_exactly(last_beginning_of_month, last_end_of_month)
    end
  end
end
```

このテストは、例えば3月1日に実行すると失敗する。常にバグを検知できるわけではないが、CIを利用することで、ずっと不具合が残り続ける可能性を減らすことができるだろう。


## 日付を外部から注入する

[「現在時刻」を外部入力とする設計と、その実装のこと - クックパッド開発者ブログ](http://techlife.cookpad.com/entry/2016/05/30/183947)

## スコープを考慮する

- before で十分なものは let(let!) にしない
 - let(let!)で定義するものは「後で参照するもの」

## 控えめなDRY

- テストはDRYにしない方が良い
- コードが重複していればDRYにして良いかというと、そうではないケースもある
  - 実例ほしい！
- 意味的に重複していればOK

### describe 外にテストデータを置かない

例えば、次のような spec があるとする

```ruby
describe 'sample specs' do
  context 'a' do
    # ...
  end

  context 'b' do
    let!(:need_in_b_and_c) { ... }        
    # ...
  end

  context 'c' do
    let!(:need_in_b_and_c) { ... }        
    # ...
  end
end
```

この場合、b と c で同じ前提条件を利用しているので、一つ上のレベルに移動してDRYにしようと考える人もいるかもしれない。

```ruby
describe 'sample specs' do
  let!(:need_in_b_and_c) { ... }        

  context 'a' do
    # ...
  end

  context 'b' do
    # ...
  end

  context 'c' do
    # ...
  end
end
```

しかし、これは良くない。'a' のコンテキストに、必要のない前提条件が含まれてしまうからだ。この例だけだとピンとこないかもしれない。このような let! が 10、context が 30 あって、どの let! がどの context に対応する前提条件なのかわからない状況を想像すると、下手に前提条件をまとめる怖さがわかるだろうか。

もちろん、すべての context において、共通で使う前提条件であれば、まとめてしまうのは問題ない。


### 各ブロックにおける前提条件の配置ルール

- 各ブロックの前提条件は、その配下のすべての expectation で利用するもののみ書く
- 特定の expectation のみで利用するものにおいては、その expectation に書く


### 落穂拾い

- 各ブロックにおける前提条件の配置ルール、の例外は次のようなケース
- だが、基本的には各ブロック内で宣言するほうが望ましい

```ruby
let!(:user) { create :user, enabled: enabled }

context 'when user is enabled' do
  let(:enabled) { true }
  it { ... }
end

context 'when user is disabled' do
  let(:enabled) { false }
  it { ... }
end

```


### 必要ないレコードを作らない

### update でデータを変更しない

### let を上書きしない

### shared_example, shared_context は控える

### subject を使うときの注意事項

### allow_any_instance_of を避ける
