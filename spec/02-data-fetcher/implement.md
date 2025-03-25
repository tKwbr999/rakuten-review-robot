# 楽天レビューボット - Edge Function によるランキング取得実装ガイド

## 1. プロジェクト概要

このプロジェクトは、楽天市場 API から女性向け商品ランキングデータを取得するシステムです。Edge Function を使用して、サーバーレスな方法でこれらの処理を実現します。

## 2. システムアーキテクチャ

システムは以下の主要コンポーネントで構成されています：

1. **データ取得モジュール**: 楽天市場 API と通信し、商品ランキングデータを取得
2. **メインモジュール**: 全体のフロー制御

## 3. 開発環境の準備

### 3.1 必要なツールとアカウント

- Deno
- 楽天デベロッパーアカウント（アプリケーション ID・アフィリエイト ID）

## 4. Edge Function の実装

### 4.1 プロジェクト構造の作成

```bash
mkdir -p rakuten-ranking-fetcher
cd rakuten-ranking-fetcher
```

### 4.2 エラー処理モジュールの実装

`errors.ts`を作成：

```typescript
export enum DataFetcherErrorType {
  NETWORK_ERROR = "NETWORK_ERROR",
  CONNECTION_ERROR = "CONNECTION_ERROR",
  TIMEOUT_ERROR = "TIMEOUT_ERROR",
  RATE_LIMIT_ERROR = "RATE_LIMIT_ERROR",
  AUTHENTICATION_ERROR = "AUTHENTICATION_ERROR",
  PARAMETER_ERROR = "PARAMETER_ERROR",
  RESPONSE_FORMAT_ERROR = "RESPONSE_FORMAT_ERROR",
  UNKNOWN_ERROR = "UNKNOWN_ERROR",
}

export class DataFetcherError extends Error {
  readonly type: DataFetcherErrorType;
  readonly statusCode?: number;
  readonly retryable: boolean;
  readonly response?: any;
  readonly cause?: Error;

  constructor(
    message: string,
    type: DataFetcherErrorType,
    options: {
      statusCode?: number;
      retryable?: boolean;
      response?: any;
      cause?: Error;
    } = {}
  ) {
    super(message);
    this.name = "DataFetcherError";
    this.type = type;
    this.statusCode = options.statusCode;
    this.retryable = options.retryable ?? this.isRetryableByDefault(type);
    this.response = options.response;
    this.cause = options.cause;
  }

  private isRetryableByDefault(type: DataFetcherErrorType): boolean {
    switch (type) {
      case DataFetcherErrorType.NETWORK_ERROR:
      case DataFetcherErrorType.CONNECTION_ERROR:
      case DataFetcherErrorType.TIMEOUT_ERROR:
      case DataFetcherErrorType.RATE_LIMIT_ERROR:
        return true;
      default:
        return false;
    }
  }
}
```

### 4.3 データ取得モジュールの実装

`data-fetcher.ts`を作成：

```typescript
import { DataFetcherError, DataFetcherErrorType } from "./errors.ts";

interface RakutenRankingOptions {
  genreId?: string;
  sex?: number;
  page?: number;
  hits?: number;
}

export interface RakutenItem {
  itemCode: string;
  itemName: string;
  itemPrice: number;
  itemUrl: string;
  shopName: string;
  mediumImageUrl: string;
  rank: number;
  reviewCount: number;
  reviewAverage: number;
}

export class RakutenRankingClient {
  private readonly baseUrl =
    "https://app.rakuten.co.jp/services/api/IchibaItem/Ranking/20170628";
  private readonly applicationId: string;
  private readonly affiliateId?: string;

  constructor(applicationId: string, affiliateId?: string) {
    this.applicationId = applicationId;
    this.affiliateId = affiliateId;
  }

  // 商品ランキング取得メソッド
  async fetchItems(options: RakutenRankingOptions): Promise<RakutenItem[]> {
    try {
      const params = new URLSearchParams({
        applicationId: this.applicationId,
        format: "json",
      });

      if (this.affiliateId) params.append("affiliateId", this.affiliateId);
      if (options.genreId) params.append("genreId", options.genreId);
      if (options.sex !== undefined)
        params.append("sex", options.sex.toString());
      if (options.page !== undefined)
        params.append("page", options.page.toString());
      if (options.hits !== undefined)
        params.append("hits", options.hits.toString());

      const url = `${this.baseUrl}?${params.toString()}`;
      const response = await fetch(url, {
        method: "GET",
        headers: { Accept: "application/json" },
      });

      if (!response.ok) {
        throw new DataFetcherError(
          `APIエラー: ${response.status} ${response.statusText}`,
          response.status === 429
            ? DataFetcherErrorType.RATE_LIMIT_ERROR
            : DataFetcherErrorType.NETWORK_ERROR,
          { statusCode: response.status }
        );
      }

      const data = await response.json();
      if (!data || !Array.isArray(data.Items)) {
        throw new DataFetcherError(
          "不正なレスポンス形式",
          DataFetcherErrorType.RESPONSE_FORMAT_ERROR
        );
      }

      return data.Items.map((item: any) => ({
        itemCode: item.Item.itemCode,
        itemName: item.Item.itemName,
        itemPrice: item.Item.itemPrice,
        itemUrl: item.Item.itemUrl,
        shopName: item.Item.shopName,
        mediumImageUrl: item.Item.mediumImageUrls[0]?.imageUrl || "",
        rank: item.Item.rank,
        reviewCount: item.Item.reviewCount || 0,
        reviewAverage: item.Item.reviewAverage || 0,
      }));
    } catch (error) {
      if (error instanceof DataFetcherError) {
        throw error;
      }
      throw new DataFetcherError(
        `データ取得エラー: ${error.message}`,
        DataFetcherErrorType.UNKNOWN_ERROR,
        { cause: error }
      );
    }
  }

  // レディースファッションデータを取得する便利メソッド
  async fetchLadiesFashionItems(
    maxItems: number = 3000
  ): Promise<RakutenItem[]> {
    const pageSize = 30; // APIの1リクエスト最大件数
    let allItems: RakutenItem[] = [];
    let page = 1;

    while (allItems.length < maxItems) {
      // レート制限に対応するため待機
      if (page > 1) {
        await new Promise((resolve) => setTimeout(resolve, 1000));
      }

      const items = await this.fetchItems({
        genreId: "100371", // レディースファッション
        sex: 2, // 女性
        page,
        hits: pageSize,
      });

      if (items.length === 0) {
        break;
      }

      allItems = [...allItems, ...items];
      page++;

      if (allItems.length >= maxItems || items.length < pageSize) {
        break;
      }
    }

    return allItems.slice(0, maxItems);
  }
}
```

### 4.4 メイン Edge Function 実装

`index.ts`を作成：

```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { RakutenRankingClient } from "./data-fetcher.ts";

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers":
    "authorization, x-client-info, apikey, content-type",
};

serve(async (req) => {
  // CORS対応
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  try {
    // 環境変数から設定を取得
    const rakutenAppId = Deno.env.get("RAKUTEN_APP_ID") || "";
    const rakutenAffiliateId = Deno.env.get("RAKUTEN_AFFILIATE_ID") || "";

    // 必須パラメータの確認
    if (!rakutenAppId) {
      return new Response(
        JSON.stringify({ error: "必要な環境変数が設定されていません" }),
        {
          headers: { ...corsHeaders, "Content-Type": "application/json" },
          status: 500,
        }
      );
    }

    // 楽天APIから商品データを取得
    const client = new RakutenRankingClient(rakutenAppId, rakutenAffiliateId);
    console.log("楽天市場から商品データの取得を開始...");

    const items = await client.fetchLadiesFashionItems(3000);
    console.log(`${items.length}件の商品データを取得しました`);

    return new Response(
      JSON.stringify({
        success: true,
        message: `${items.length}件の商品データを取得しました`,
        items: items,
      }),
      {
        headers: { ...corsHeaders, "Content-Type": "application/json" },
      }
    );
  } catch (error) {
    console.error("エラーが発生しました:", error);

    return new Response(
      JSON.stringify({
        error: error.message,
        success: false,
        type: error.type || "UNKNOWN_ERROR",
        details: error.stack,
      }),
      {
        headers: { ...corsHeaders, "Content-Type": "application/json" },
        status: 500,
      }
    );
  }
});
```

## 5. デプロイと設定

### 5.1 環境変数設定

```bash
export RAKUTEN_APP_ID="あなたの楽天アプリケーションID"
export RAKUTEN_AFFILIATE_ID="あなたの楽天アフィリエイトID"
```

### 5.2 Edge Function の実行

```bash
deno run --allow-net --allow-env index.ts
```

## 6. トラブルシューティング

### 6.1 一般的な問題

- **レート制限エラー**: 待機時間の調整
- **タイムアウト**: 処理の分割

### 6.2 デバッグ方法

- ログの確認
- ローカルでのテスト実行: `deno run --allow-net --allow-env --watch index.ts`

## まとめ

この手順に従って実装することで、Edge Function を利用した楽天商品ランキングデータ取得システムを構築できます。レート制限対応、エラーハンドリングなどを適切に行い、安定した運用ができるようにしてください。
