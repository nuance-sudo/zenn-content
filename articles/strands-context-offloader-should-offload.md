---
title: "Strands Agentsのshould_offloadでツール結果を選択的にオフロードする"
emoji: "📦"
type: "tech"
topics: ["context", "rag", "agent", "aws", "strandsagent"]
published: false
---

## はじめに

エージェントが扱うツール結果には、表データの一括取得や説明資料の全文など、さまざまな種類があります。どちらも大きな結果になり得ますが、回答に必要なのが一部の行だけの場合もあれば、資料全体を会話に残して参照したい場合もあります。

Strands AgentsのContext Offloaderは、大きなツール結果を外部ストレージへ退避し、必要な部分だけを再取得できるプラグインです。従来はtoken数の閾値を超えた結果が一律で退避されていましたが、v1.51で`should_offload`コールバックが追加され、退避するかどうかを柔軟に判断できるようになりました。

本記事では、表形式のrawデータだけを退避し、PDFから抽出した説明資料は会話に残すPoCを作り、実モデルでその動作と入力token数を確認します。

## Context Offloaderとは

Context Offloaderは、Strands Agents Python v1.38.0で追加されたプラグインです（[v1.38.0リリースノート](https://github.com/strands-agents/harness-sdk/releases/tag/python/v1.38.0)）。概要はこちらの記事がわかりやすいです（[Strands AgentsのContext Offloaderプラグインを試してみた](https://dev.classmethod.jp/articles/strands-agents-context-offloader/)）。

やっていることはシンプルで、**大きいツール結果を外部ストレージへ退避し、会話履歴にはpreview（先頭の要約）と再取得用の参照だけを残します**。エージェントが退避されたデータを必要とする場合は、`retrieve_offloaded_content`というツールでpatternや行範囲を指定して、必要な断片だけを取り出せます。

## should_offloadコールバック

従来の動作では、ツール結果が`max_result_tokens`の閾値を超えると、一律で外部ストレージへ退避していました。つまり、閾値を超えた結果はツールの種類に関係なくすべて退避されます。

```python
# v1.38.0〜: 閾値を超えた結果はすべて退避
offloader = ContextOffloader(
    storage=LocalFileStorage("./offloads"),
    max_result_tokens=200,
)
```

v1.51で`should_offload`コールバックが追加されました。

プラグインが`max_result_tokens`でトークン量を判定したあとに`should_offload`が呼ばれ、最終的に退避するかをより柔軟に決められます。

```python
# v1.51〜: 閾値を超えた結果のうち、退避対象を自分で選べる
def should_offload(tool_name: str, token_count: int, **_: object) -> bool:
    return tool_name == "export_customer_rows"

offloader = ContextOffloader(
    storage=LocalFileStorage("./offloads"),
    max_result_tokens=200,
    should_offload=should_offload,
)
```

これにより、以下のような制御ができます。

- **DB取得結果** → 退避して、必要な範囲だけ再取得
- **ナレッジベース検索・集計結果** → ある程度サイズが大きくても会話コンテキストに残す

## 今回検証する使い分け

Context Offloaderを使う目的は、単に大きい結果をすべて会話の外へ出すことではありません。今回のPoCでは、同じく閾値を超える結果でも、その後の会話での使われ方に応じて扱いを分けます。

- **表形式のrawデータ**: 回答に必要なのは通常、一部の行や値だけです。全行を会話に残さず、外部へ退避してから必要な範囲だけを取り出します
- **説明資料の本文**: 定義や前提、複数箇所の記述を回答に使う可能性があります。今回のPoCでは、PDFから抽出したテキストを会話に残します

この使い分けを試すため、次の3つのツールを用意しました。

- **少量取得ツール**: 指定月の少数行を返す
- **raw exportツール**: 指定期間の表データをJSONでまとめて返す
- **資料検索ツール**: 指標の定義を説明するPDFの抽出テキストを返す

`should_offload`ではraw exportツールだけを退避対象にします。そして実モデルを使い、500行の表データが退避され、必要な値だけを再取得して回答できるかを確認します。あわせて、閾値を超えるPDF結果が退避されず、その内容を使って回答できるかを別の質問で確認します。

### 検証データと実行条件

表データには、食料、光熱・水道、交通・通信など10カテゴリの名目値と実質値を持つ、250カ月分、合計5,000行の合成データを使います。画面では多数の値が並んでいますが、質問への回答に必要なのは、そのうちの特定の月・カテゴリの値です。

![PoCで使用する月次表データの例。多数の行とカテゴリのうち、回答に必要な値は一部に限られる](/images/strands-context-offloader-should-offload/table-data.png)

*図1: 合成データの項目設計で参考にしたe-Statの基本系列。PoCでは、この項目構成を模した合成データを使用しています。参考: [e-Stat「世帯消費動向指数 1-1-1 基本系列（原数値）」](https://www.e-stat.go.jp/stat-search/files?page=1&layout=datalist&toukei=00200567&tstat=000001157746&cycle=0&tclass1=000001157748&stat_infid=000032147336&tclass2val=0)*

説明資料には、6ページの「世帯消費動向指数（CTI）」概要PDFを使います。抽出テキストは13,173文字あり、今回設定した閾値を超えます。一方で、指標の定義や作成方法を説明する質問では、資料中の複数の記述が回答の根拠になります。

![PoCで使用する世帯消費動向指数の概要PDF。指標の概要、グラフ、表が掲載されている](/images/strands-context-offloader-should-offload/cti-overview.png)

*図2: PoCで資料検索結果として返す6ページのPDF。今回は抽出テキストを会話に残します。出典: 総務省統計局「消費動向指数（CTI）概要」([e-Stat掲載ページ](https://www.e-stat.go.jp/stat-search/files?page=1&layout=datalist&toukei=00200567&tstat=000001157746&cycle=0&tclass1=000001157748&stat_infid=000032147335&tclass2val=0))*

| 要素 | 内容 | Context Offloaderでの扱い |
| --- | --- | --- |
| 表データ | 5,000行の合成データ | 大量取得時は退避 |
| 説明資料 | 6ページ、抽出テキスト13,173文字のPDF | 会話に残す |
| ストレージ | `LocalFileStorage` | ローカルファイルへ保存 |
| モデル | Amazon Bedrock Claude Haiku 4.5 | US inference profileを使用 |
| 閾値 | `max_result_tokens=200`、`preview_tokens=50` | 両方の大きい結果が閾値を超える設定 |

### raw exportだけを退避対象にする

今回のPoCでは、`should_offload`でraw exportツールだけを退避対象にします。

```python
RAW_EXPORT_TOOL = "export_cti_raw"

def should_offload(tool_name: str, token_count: int, **_: object) -> bool:
    return tool_name == RAW_EXPORT_TOOL
```

この設定では、閾値を超えた結果のうち`export_cti_raw`だけが退避されます。PDFを返す`search_gaiyo_knowledge`も閾値を超えますが、コールバックが`False`を返すため会話履歴に残ります。

## 検証結果

Claude Haiku 4.5で、表データとPDFを使う質問を別々に実行しました。どちらも`max_tokens=512`とし、表質問では500行のrawデータ、PDF質問では6ページから抽出した13,173文字を入力データとして使いました。

### 表質問：500行は退避し、必要な部分だけを戻す

「2002年1月の実質『光熱・水道』の値」を質問すると、次の順に処理されました。

1. **500行を退避する**

   エージェントが`export_cti_raw`を呼び出すと、返された500行はローカルファイルへ退避されました。会話履歴には、結果の先頭を示すpreviewと保存先の参照だけが残ります。previewの一部は次のとおりです。

   ```text
   [Offloaded: 1 blocks, ...]

   {
     "items": [
       {
         "period": "2002-01",
         "basis": "real",
         "category": "食料",
         "value": 90.0
       },
       ...
     ]
   }

   [Stored references]
   tooluse_cHHIBB56tQN6Vm70d75oty_0
   ```

   この時点では、質問対象の「光熱・水道」はpreviewに含まれていません。

2. **必要なデータを検索する**

   エージェントは保存先の参照を使い、`retrieve_offloaded_content`で「光熱・水道」を検索しました。

   ```json
   {
     "reference": "tooluse_cHHIBB56tQN6Vm70d75oty_0",
     "pattern": "光熱・水道"
   }
   ```

3. **取得した値を回答する**

   限定取得した結果には、次のレコードが含まれていました。表示項目だけを抜粋しています。

   ```json
   {
     "period": "2002-01",
     "basis": "real",
     "category": "光熱・水道",
     "value": 91.3
   }
   ```

   500行すべてをモデルへ戻すのではなく、このレコードを使って回答しました。

   > 2002年1月の実質「光熱・水道」の値は**91.3**です。

同じ500行を使った対照実験では、Context Offloaderを使わない場合と使う場合で、モデル呼び出しごとの入力token数は次のようになりました。

| 条件 | モデル呼び出しごとの入力token数 | ピーク入力token数 | 最終回答 |
| --- | --- | ---: | --- |
| Context Offloaderなし | 1,179 → 68,634 | 68,634 | `91.3` |
| Context Offloaderあり | 1,923 → 2,553 → 2,726 → 3,185 | 3,185 | `91.3` |

この1回の対照実験では、ピーク入力は68,634 tokenから3,185 tokenへ減り、差は65,449 token、割合では95.4%でした。

### PDF質問：抽出テキスト全体を会話に残す

「世帯消費動向指数は何を表し、どの統計を合成して作られている？」と質問すると、エージェントは`search_gaiyo_knowledge`を1回呼び出しました。

このツールが返すのは、「消費動向指数（CTI）概要」全6ページから抽出した13,173文字、約3,293 tokenのテキストです。`max_result_tokens=200`を超えていますが、`should_offload`の対象外なので退避されません。表質問とは異なり、抽出テキスト全体が会話履歴に残った状態で回答が生成されました。

最終回答は次のとおりです。

> 検索結果から、世帯消費動向指数についての情報を得ました。PDFの内容に基づいて、以下の2つの要点をお答えします。
>
> - **指標**：世帯消費動向指数（Household Consumption Trend Index）は、世帯における平均消費支出額について、基準年（2020年）の平均月額を100として表す指数です。
> - **合成元**：家計調査の結果に、家計消費状況調査及び家計消費単身モニター調査の結果を合成した金額によって作成されています。

2つの結果から、同じ閾値を超えるツール結果でも、表のrawデータは退避して必要部分だけを戻し、説明資料は全文を会話に残す、という使い分けを実モデルでも確認できました。

## まとめ：should_offloadで何が便利になったか

Context Offloaderが`should_offload`により、閾値を超えたツール結果を退避するかどうかを、ツール名やtoken数に応じて柔軟に判断できるようになりました。

これにより、必要な部分だけを参照したいデータと、全体を会話に残したいデータを分けて扱えるようになりました。
