# 結論から言うと？
- **OpenAI APIのサーチモデル(*-search-preview)で仕様変更があったよ**
- **システムプロンプト(`role`が`system`)が効かなくなったよ**
- **ユーザークエリ(`role`が`user`)に指示も含む必要があるよ**

# バージョン情報
openai 1.74.0(最新)

# 検証
## コード
``` python
from openai import OpenAI
client = OpenAI(api_key = 'OPENAI_API_KEY')

def openai_search(system_prompt=None,user_query=None):

    # 検証用の分岐
    if system_prompt and user_query: # システムプロンプトとユーザークエリを指定
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_query}
            ]
    elif system_prompt: # システムプロンプトのみを指定
        messages = [
            {"role": "system", "content": system_prompt},
            ]
    else: # ユーザークエリのみを指定
        messages = [
            {"role": "user", "content": user_query}
            ]
    
    # 検索オプション
    options = {
        "search_context_size": "high",
        "user_location": {
            "type": "approximate",
            "approximate": {
                "country": "JP",
            },
        },
    }

    # レスポンス
    response = client.chat.completions.create(
        model='gpt-4o-search-preview',
        web_search_options=options,
        messages=messages
    )

    return response.choices[0].message.content
```
## 【動かない例】システムプロンプトのみを指定した場合
``` python
print(openai_search(system_prompt="今日の東京の天気を教えてください"))
```
**システムプロンプトのみの指定だと、全く関係のない出力になる。**クエリが指定されていない状態。
``` shell
日本の所得税の申告期間は通常、毎年2月16日から3月15日までです。しかし、2025年は3月15日が土曜日に当たるため、申告期限は翌営業日の3月17日（月曜日）に延長されました。したがって、2024年分の所得税および贈与税の申告と納付の期限は2025年3月17日となります。 ([news.bloombergtax.com](https://news.bloombergtax.com/daily-tax-report-international/japan-tax-agency-announces-filing-payment-deadlines-for-various-taxes-1?utm_source=openai))

また、個人事業主の消費税の申告と納付の期限は2025年3月31日です。 ([news.bloombergtax.com](https://news.bloombergtax.com/daily-tax-report-international/japan-tax-agency-announces-filing-payment-deadlines-for-various-taxes-1?utm_source=openai))

申告方法としては、税務署への直接提出、郵送、または国税庁の電子申告システム「e-Tax」を利用することができます。e-Taxは24時間利用可能で、忙しい方にも便利な方法です。 ([mailmate.jp](https://mailmate.jp/blog/how-to-file-tax-return-in-japan?utm_source=openai))

期限内に申告を行わない場合、延滞税や無申告加算税などのペナルティが課される可能性があります。そのため、期限を守って申告・納付を行うことが重要です。 ([mailmate.jp](https://mailmate.jp/blog/how-to-file-tax-return-in-japan?utm_source=openai))
```

## 【動かない例】システムプロンプトとユーザークエリを指定した場合
``` python
print(openai_search(system_prompt="語尾に「ござ」をつけて",user_query="今日の東京の天気を教えて"))
```
**システムプロンプトは無視してユーザークエリのみに従う**様子
``` shell
本日、4月15日（火）の東京都江戸川区中葛西6丁目13の天気予報は以下の通りです。

## Weather for 〒134-0083 東京都江戸川区中葛西6丁目13:
現在の状態: 所により晴れ、59°F (15°C)

Daily Forecast:
* Tuesday, April 15: 低: 49°F (10°C)、高: 69°F (21°C)、説明: とても風が強い
* Wednesday, April 16: 低: 52°F (11°C)、高: 69°F (21°C)、説明: 晴れ
* Thursday, April 17: 低: 56°F (13°C)、高: 70°F (21°C)、説明: 晴れ
* Friday, April 18: 低: 61°F (16°C)、高: 72°F (22°C)、説明: おおむね曇り
* Saturday, April 19: 低: 61°F (16°C)、高: 75°F (24°C)、説明: 晴れ
* Sunday, April 20: 低: 56°F (13°C)、高: 68°F (20°C)、説明: おおむね曇り
* Monday, April 21: 低: 56°F (14°C)、高: 77°F (25°C)、説明: より暖かい

Severe Weather Alerts:
* 江戸川区陸上: 強風注意報は、午後9:00 (JST)まで続く見込みです。発表者: 気象庁、、開始時刻: Tuesday, April 15, 03:00 AM、終了時刻: Tuesday, April 15, 09:00 PM
* 江戸川区: 波浪注意報は、午後9:00 (JST)まで続く見込みです。発表者: 気象庁、、開始時刻: Tuesday, April 15, 03:00 AM、終了時刻: Tuesday, April 15, 09:00 PM
* 江戸川区: 雷注意報は、午後9:00 (JST)まで続く見込みです。発表者: 気象庁、、開始時刻: Tuesday, April 15, 03:00 AM、終了時刻: Tuesday, April 15, 09:00 PM


現在、強風注意報、波浪注意報、雷注意報が発表されており、いずれも午後9時（JST）まで続く見込みです。

強風や高波、雷に注意し、安全にお過ごしください。
```
## 【対処法】ユーザープロンプトに指示を含めましょう
``` python
print(openai_search(user_query="今日の天気を教えて。語尾に「ござ」をつけて"))
```
きちんと指示に従ってくれました。
``` shell
本日、4月15日（火）の東京都江戸川区中葛西6丁目13の天気予報でござ。

## Weather for 〒134-0083 東京都江戸川区中葛西6丁目13:
現在の状態: おおむね曇り、63°F (17°C)

Daily Forecast:
* Tuesday, April 15: 低: 49°F (10°C)、高: 69°F (21°C)、説明: とても風が強い
* Wednesday, April 16: 低: 52°F (11°C)、高: 69°F (21°C)、説明: 晴れ
* Thursday, April 17: 低: 56°F (13°C)、高: 70°F (21°C)、説明: 晴れ
* Friday, April 18: 低: 61°F (16°C)、高: 72°F (22°C)、説明: おおむね曇り
* Saturday, April 19: 低: 61°F (16°C)、高: 75°F (24°C)、説明: 晴れ
* Sunday, April 20: 低: 56°F (13°C)、高: 68°F (20°C)、説明: おおむね曇り
* Monday, April 21: 低: 56°F (14°C)、高: 77°F (25°C)、説明: より暖かい

Severe Weather Alerts:
* 江戸川区陸上: 強風注意報は、午後9:00 (JST)まで続く見込みです。発表者: 気象庁、、開始時刻: Tuesday, April 15, 03:00 AM、終了時刻: Tuesday, April 15, 09:00 PM
* 江戸川区: 波浪注意報は、午後9:00 (JST)まで続く見込みです。発表者: 気象庁、、開始時刻: Tuesday, April 15, 03:00 AM、終了時刻: Tuesday, April 15, 09:00 PM
* 江戸川区: 雷注意報は、午後9:00 (JST)まで続く見込みです。発表者: 気象庁、、開始時刻: Tuesday, April 15, 03:00 AM、終了時刻: Tuesday, April 15, 09:00 PM


現在の天気はおおむね曇りで、気温は17℃でござ。本日は非常に風が強く、最高気温は21℃、最低気温は10℃の予想でござ。また、強風注意報、波浪注意報、雷注意報が発表されており、いずれも午後9時まで続く見込みでござ。

外出の際は、強風や雷に十分ご注意くださいませ。
```