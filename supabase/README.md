# Supabase Edge Function 実装ガイド - 楽天レビューボット

## 概要

このガイドでは、Supabase Edge Functions を使用して楽天市場 API から女性向け商品ランキングデータを取得する実装手順を段階的に説明します。

## 前提条件

- Supabase アカウントとプロジェクト
- 楽天デベロッパーアカウント（アプリケーション ID・アフィリエイト ID）
- Deno（ローカル開発用）
- Supabase CLI

## 実装ステップ

### ステップ 1: 環境セットアップ

1. **CLI のインストールと設定**

   ```bash
   npm install -g supabase
   supabase login
   supabase link --project-ref <あなたのプロジェクト参照ID>
   ```

2. **Edge Function の作成**
   ```bash
   cd supabase
   supabase functions new data-fetcher
   ```

### ステップ 2: エラー処理モジュールの実装

1. `supabase/functions/data-fetcher/errors.ts` を作成:

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

### ステップ 3: データ取得モジュールの実装

1. `supabase/functions/data-fetcher/data-fetcher.ts` を作成:

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

### ステップ 4: メイン Edge Function 実装

1. `supabase/functions/data-fetcher/index.ts` を作成:

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
       const client = new RakutenRankingClient(
         rakutenAppId,
         rakutenAffiliateId
       );
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

### ステップ 5: 環境変数の設定とテスト実行

1. **ローカルでのテスト用 .env ファイルの作成**
   `.env` ファイルをプロジェクトルートに作成し、以下の内容を追加:

   ```
   RAKUTEN_APP_ID=あなたの楽天アプリケーションID
   RAKUTEN_AFFILIATE_ID=あなたの楽天アフィリエイトID
   ```

2. **ローカルでの実行**

   ```bash
   supabase functions serve data-fetcher --env-file .env
   ```

3. **動作確認**
   ```bash
   curl http://localhost:54321/functions/v1/data-fetcher
   ```

### ステップ 6: Supabase にデプロイ

1. **シークレットの設定**

   ```bash
   supabase secrets set RAKUTEN_APP_ID=あなたの楽天アプリケーションID
   supabase secrets set RAKUTEN_AFFILIATE_ID=あなたの楽天アフィリエイトID
   ```

2. **デプロイの実行**

   ```bash
   supabase functions deploy data-fetcher
   ```

3. **本番環境でのテスト**
   ```bash
   curl https://<プロジェクトID>.supabase.co/functions/v1/data-fetcher
   ```

## トラブルシューティング

### 一般的な問題

- **レート制限エラー**:

  - 待機時間を 1 秒から増やす（例: 2 秒）
  - エラー時のバックオフ戦略を実装

- **タイムアウト**:
  - データ取得を複数回に分割
  - 最大取得アイテム数を減らす

### ログの確認

```bash
supabase functions logs data-fetcher
```

## 注意点

- Edge Functions は実行時間に制限があります（デフォルトで 10 秒）
- 大量データを取得する場合は、複数回の呼び出しに分割することを検討
- 楽天 API のレート制限（1 リクエスト/秒）を遵守するよう設計されています

## 参考リソース

- [Supabase Edge Functions ドキュメント](https://supabase.com/docs/guides/functions)
- [楽天ウェブサービス API ドキュメント](https://webservice.rakuten.co.jp/documentation/ichiba-item-ranking)
- [Deno ドキュメント](https://deno.land/manual)
