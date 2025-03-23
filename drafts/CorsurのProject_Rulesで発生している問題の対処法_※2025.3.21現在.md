# この記事の対象者
この記事は以下のような方を対象としています：

- Cursorの最新バージョン（v0.47）を使用している
- Cursorで**Project Rulesを設定したものの、適用されなくて困っている**
- **AgentにProject Rulesファイルを作成させたら、空のファイルになって困っている**（※先にお伝えしておくと、現時点では根本的な解決はしていません）

## Project Rules（知っている人は読み飛ばし推奨）
![](https://storage.googleapis.com/zenn-user-upload/6aa99850a380-20250321.png)
v0.45から追加された機能で、プロジェクトごとにAgentの動作をカスタムするためのルールを設定できる機能です。

プロジェクトのルートディレクトリに`.cursor/rules`ディレクトリを作成し、ルールを記述した`.mdcファイル`を配置することで使用できます。


::: message
以前は`.cusorrules`ファイルが使用されていましたが、現在は非推奨となっています。
Project Rulesを使用しましょう。
:::

### ルール定義ファイル（.mdc）の構成
以下のfrontmatter部分と、markdown形式で記述したルールコンテキストで構成されます。

```
---
`description`: ルールが適用される条件の説明
`globs`: ルールが適用されるファイル/フォルダのglobパターンを記述(ex. src/**/*)
`alwaysApply`: 常にルールを適用するかどうかのbool値
---
```

### Rule Type
![](https://storage.googleapis.com/zenn-user-upload/e498a2746e7c-20250321.png)
v0.47.5から、Project RulesにRule Typeが追加されました。
これにより、作成したProject Rulesがいつ適用されるかを明確に指定できるようになりました。

**使用できるルールタイプ**
- `Always Apply`: ルールを常時適用する
- `Auto Attached`: 指定したglobパターンに一致するファイルが参照された時に自動的にルールが適用される
- `Manual Reference`: ルールファイルを明示的に参照した場合に適用
- `Semantic Selectio`: Agentがdescriptionを読んで、現在のタスクに関連すると判断した場合に適用

# 現バージョンで発生している問題
主に以下の2つの問題が発生しているみたいです。
- **「Auto Attached」の自動参照が機能しない**
- **Agentにルール定義ファイル(.mdc)を作成させると中身が空になる**
  
# 「Auto Attached」の自動参照が機能しない問題について
ルールタイプに`Auto Attached`を選択した場合、`File pattern matches`に適切なglobパターンを入力することで、本来は対象ファイルが参照されているAgentとのチャットでは自動でルールが適用されます。しかし、現状はこれが正常に機能していないようです。

実際、以下のように`test.mdc`を作成して
![](https://storage.googleapis.com/zenn-user-upload/44156f354bc3-20250321.png)

Agentにてtestフォルダを参照して
「現在、あなたにはどのようなルールが適用されていますか？」
と質問してみましたが、`test.mdc`の内容は参照されていないようです。
![](https://storage.googleapis.com/zenn-user-upload/a5c5b5d933a9-20250321.png)

この問題は以下のCursorのフォーラムでも言及されています。
https://forum.cursor.com/t/no-project-rules-found/47245

## 原因

**`Auto Attached`では`description`フィールドがUI上に入力欄が表示されないにもかかわらず、
必須なため**

です。
なかなかひどい話だと思うのですが、最新のv0.47ではルールタイプに`Auto Attached`を選択した場合は、**Descriptionの入力欄が表示されない**にもかかわらず、これがないと**システムがルール定義ファイルを認識できない**ようです。
(推測ですが、システムがルールファイルをインデックス化または認識する際に、このフィールドの内容を利用していると思われます)

## 対策
**メモ帳で編集して`Description`の内容を追加する**

Cursorで`.mdc`ファイルを開くと、自動でルール定義ファイル用のUIが開かれてしまうため、メモ帳などのテキスト編集アプリを使用して、`frontmatter`に`Description`を追加することで、システムが認識できるようになります。

さっそくやってみましょう。
`.cursor/rules/test.mdc`をメモ帳で開いて、`description`に説明を追加します。
![](https://storage.googleapis.com/zenn-user-upload/b547e5141c11-20250321.png)

改めてAgentに現在適用されているルールを質問します。
![](https://storage.googleapis.com/zenn-user-upload/1586e8e94b91-20250321.png)

テストルールについて言及されましたね。以上です。

# Agentにルール定義ファイル(.mdc)を作成させると中身が空になる問題について

Project Rulesを使用していると、以下のようなニーズから**ルール定義ファイルをAgent自身に作成させたい**といったニーズが出てくるかと思います。
- Agentにそこまでのチャットの作業手順をルール化して保存させたい
- 使用するライブラリについて、使い方をルール化して保存させたい

参考：
https://zenn.dev/caphtech/articles/cursor-memory-function

しかし、現在のバージョンでこれを行うと、Agentに作成させたルール定義ファイル(.mdc)の中身が空になってしまうという問題があるようです。

この件は以下のフォーラムでも言及されています。
https://forum.cursor.com/t/cursor-agent-cannot-write-to-cursor-rules-files/61732

## 原因
