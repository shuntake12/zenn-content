---
title: "今日の市場ブリーフィング"
emoji: "📊"
type: "tech"
topics: ["finance", "investment", "economy", "data", "ai"]
published: true
---

## この記事で分かること

- **yfinance等の外部APIで市場データ取得が失敗する原因と構造的リスク**
- **誤った数値の自動補填やデータの捏造を防ぐ不整合検証の実装方法**
- **外部データソース障害時にもパイプラインを止めないフォールバック設計**

## 金融APIのデータ取得が予告なく停止する理由

外部の市場データ取得処理（特に`yfinance`等のライブラリ）は、事前の通知なく突然失敗します。

なぜなら、`yfinance`はYahoo! Financeの非公式なエンドポイントやWebページ構造に依存してデータを取得しているため、仕様変更やスクレイピング対策の強化によってリクエストが遮断されるからです。Python等でシステムを自動化している場合、コード自体に変更がなくても、ある日突然空のデータ（`Empty DataFrame`）が返される現象が発生します。

データ取得の失敗は単なるネットワークエラーではなく、日常的に発生する前提としてアーキテクチャを設計することが重要です。

:::details yfinanceでデータ取得が失敗する代表的なパターン
- API提供元によるレート制限（HTTP 429 Too Many Requests）の適用
- 配信元のHTML/JSON構造変更に伴うデータパーサーの不具合
- 短時間での大量リクエストによるIPアドレスの制限
:::

## データ捏造・誤分析を防ぐバリデーションロジックを組む方法

市場データが取得できなかった場合に、過去データの自動補填（フォワードフィル）や推計値の挿入を行ってはなりません。

実務においては、間違った数値を出力して歪んだ分析結果を提供するリスクの方が、データ不達でシステム処理を停止させるリスクよりも遥かに高いからです。数値の捏造や不正確な推計を防ぐため、データ取得ロジックの直後に厳格な存在チェックとスキーマ検証を実装する必要があります。

```python
import yfinance as yf
import pandas as pd

def fetch_market_data(ticker_symbol: str) -> pd.DataFrame:
    ticker = yf.Ticker(ticker_symbol)
    df = ticker.history(period="1d")
    
    # データが空、または必要なカラムが存在しない場合は即座に例外を投げる
    if df.empty or 'Close' not in df.columns:
        raise ValueError(f"市場データの取得に失敗しました: {ticker_symbol}")
    
    # 異常値や欠損値（NaN）の検知
    if df['Close'].isnull().any():
        raise ValueError(f"不完全なデータが含まれています: {ticker_symbol}")
        
    return df
```

:::message
データ取得失敗時に、デフォルト値（0や前日終値）を自動生成して後続処理を続行させる実装は危険です。データ不達時は処理を停止し、データが存在しない旨を明示する設計に統一してください。
:::

## 金融データ取得を堅牢にするマルチソース構成の実装方法

単一のデータソースに依存する設計は、システム全体の単一障害点（SPOF）となります。

プライマリのAPI障害時にセカンダリのAPIへ自動で切り替わるフォールバック処理を構築することで、市場データの取得成功率を大幅に引き上げることができます。実務では、`yfinance`のような無料ソースに加え、`Alpha Vantage`や`Polygon.io`などの予備APIを組み合わせる構成が一般的です。

以下のコードは、プライマリ処理が失敗した際にフォールバック用関数を呼び出す基本的なパターンを示しています。

```python
def get_market_data_with_fallback(ticker: str) -> pd.DataFrame:
    # プライマリデータソース（yfinance）からの取得
    try:
        return fetch_from_yfinance(ticker)
    except Exception as primary_error:
        print(f"プライマリAPIからの取得に失敗しました。フォールバックを実行します: {primary_error}")
    
    # セカンダリデータソース（Alpha Vantage等）からの取得
    try:
        return fetch_from_secondary_api(ticker)
    except Exception as secondary_error:
        # すべてのソースで取得失敗した場合はログを出力して安全に終了する
        raise RuntimeError("すべてのデータソースからの取得に失敗しました。データ非表示で処理を完了します。")
```

## まとめ

- **yfinanceなどの非公式APIは仕様変更や制限により突然停止するリスクがある**
- **データの誤分析を防ぐため、取得失敗時に推定値を補填せず明示的にエラーハンドリングを行う**
- **マルチソース構成を導入し、プライマリAPI停止時もセカンダリへ自動切り替えを行う**

次に何をすべきか：運用中のデータ取得スクリプトにおいて、APIレスポンスが空の場合に即座に処理を中断するチェック処理が実装されているかコードレビューを実施してください。
