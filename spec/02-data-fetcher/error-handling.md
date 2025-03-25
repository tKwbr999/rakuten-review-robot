# エラー処理仕様

## 1. 概要

本ドキュメントは、楽天市場APIからのデータ取得時に発生する可能性のあるエラーの処理方針を定義します。システムの安定性と回復力を確保するため、様々なエラー状況に対する対応策を詳細に記述します。

## 2. エラーの種類と対応方針

### 2.1 エラーカテゴリ

データ取得モジュールでは、以下のカテゴリのエラーが発生する可能性があります：

| エラーカテゴリ | 説明 | 重大度 |
|-------------|------|--------|
| ネットワークエラー | API接続や通信に関するエラー | 高 |
| レート制限エラー | APIのリクエスト制限超過によるエラー | 中 |
| 認証エラー | APIキー不正などの認証失敗 | 高 |
| パラメータエラー | リクエストパラメータの不正 | 中 |
| レスポンスエラー | 不正な形式のレスポンス | 中 |
| タイムアウトエラー | リクエスト処理の時間超過 | 高 |
| 内部エラー | モジュール内部の処理エラー | 高 |

### 2.2 エラーコードと詳細定義

```typescript
// エラータイプ定義
enum DataFetcherErrorType {
  // ネットワークエラー
  NETWORK_ERROR = "NETWORK_ERROR",                  // 一般的なネットワークエラー
  CONNECTION_ERROR = "CONNECTION_ERROR",            // 接続エラー
  TIMEOUT_ERROR = "TIMEOUT_ERROR",                  // タイムアウトエラー
  
  // レート制限エラー
  RATE_LIMIT_ERROR = "RATE_LIMIT_ERROR",            // レート制限超過
  THROTTLING_ERROR = "THROTTLING_ERROR",            // スロットリングによるエラー
  
  // 認証エラー
  AUTHENTICATION_ERROR = "AUTHENTICATION_ERROR",    // 認証エラー
  INVALID_API_KEY = "INVALID_API_KEY",              // 不正なAPIキー
  
  // パラメータエラー
  PARAMETER_ERROR = "PARAMETER_ERROR",              // 一般的なパラメータエラー
  INVALID_GENRE_ID = "INVALID_GENRE_ID",            // 不正なジャンルID
  INVALID_PAGE_NUMBER = "INVALID_PAGE_NUMBER",      // 不正なページ番号
  
  // レスポンスエラー
  RESPONSE_FORMAT_ERROR = "RESPONSE_FORMAT_ERROR",  // レスポンス形式エラー
  DATA_PARSING_ERROR = "DATA_PARSING_ERROR",        // データパースエラー
  
  // システムエラー
  SYSTEM_ERROR = "SYSTEM_ERROR",                    // システムエラー
  UNKNOWN_ERROR = "UNKNOWN_ERROR"                   // 未分類のエラー
}
```

### 2.3 カスタムエラークラス

```typescript
class DataFetcherError extends Error {
  readonly type: DataFetcherErrorType;
  readonly statusCode?: number;
  readonly retryable: boolean;
  readonly response?: any;
  
  constructor(
    message: string, 
    type: DataFetcherErrorType, 
    options: {
      statusCode?: number,
      retryable?: boolean,
      response?: any,
      cause?: Error
    } = {}
  ) {
    super(message, { cause: options.cause });
    this.name = "DataFetcherError";
    this.type = type;
    this.statusCode = options.statusCode;
    this.retryable = options.retryable ?? this.isRetryableByDefault(type);
    this.response = options.response;
  }
  
  // エラータイプに基づくデフォルトのリトライ可否判定
  private isRetryableByDefault(type: DataFetcherErrorType): boolean {
    switch (type) {
      case DataFetcherErrorType.NETWORK_ERROR:
      case DataFetcherErrorType.CONNECTION_ERROR:
      case DataFetcherErrorType.TIMEOUT_ERROR:
      case DataFetcherErrorType.RATE_LIMIT_ERROR:
      case DataFetcherErrorType.THROTTLING_ERROR:
        return true;
      
      case DataFetcherErrorType.AUTHENTICATION_ERROR:
      case DataFetcherErrorType.INVALID_API_KEY:
      case DataFetcherErrorType.PARAMETER_ERROR:
      case DataFetcherErrorType.INVALID_GENRE_ID:
      case DataFetcherErrorType.INVALID_PAGE_NUMBER:
      case DataFetcherErrorType.RESPONSE_FORMAT_ERROR:
      case DataFetcherErrorType.DATA_PARSING_ERROR:
      case DataFetcherErrorType.SYSTEM_ERROR:
      case DataFetcherErrorType.UNKNOWN_ERROR:
        return false;
    }
  }
}
```

## 3. エラー検出と生成

### 3.1 エラー検出ロジック

各種エラー状況を検出するロジックを定義します：

```typescript
// APIレスポンスからエラーを検出
function detectAPIError(response: Response, data: any): DataFetcherError | null {
  // HTTP状態コードによるエラー検出
  if (!response.ok) {
    if (response.status === 401 || response.status === 403) {
      return new DataFetcherError(
        "API認証エラー：APIキーを確認してください", 
        DataFetcherErrorType.AUTHENTICATION_ERROR,
        { statusCode: response.status, response: data }
      );
    }
    
    if (response.status === 429) {
      return new DataFetcherError(
        "レート制限を超過しました", 
        DataFetcherErrorType.RATE_LIMIT_ERROR,
        { statusCode: response.status, response: data, retryable: true }
      );
    }
    
    if (response.status === 400) {
      // エラーメッセージからより詳細なエラータイプを判定
      if (data?.error === "wrong_parameter") {
        if (data?.error_description?.includes("genreId")) {
          return new DataFetcherError(
            `不正なジャンルID: ${data.error_description}`, 
            DataFetcherErrorType.INVALID_GENRE_ID,
            { statusCode: response.status, response: data }
          );
        }
        if (data?.error_description?.includes("page")) {
          return new DataFetcherError(
            `不正なページ番号: ${data.error_description}`, 
            DataFetcherErrorType.INVALID_PAGE_NUMBER,
            { statusCode: response.status, response: data }
          );
        }
        return new DataFetcherError(
          `パラメータエラー: ${data.error_description}`, 
          DataFetcherErrorType.PARAMETER_ERROR,
          { statusCode: response.status, response: data }
        );
      }
    }
    
    if (response.status >= 500) {
      return new DataFetcherError(
        `楽天市場APIサーバーエラー: ${response.status} ${response.statusText}`, 
        DataFetcherErrorType.SYSTEM_ERROR,
        { statusCode: response.status, response: data, retryable: true }
      );
    }
    
    // その他のHTTPエラー
    return new DataFetcherError(
      `APIエラー: ${response.status} ${response.statusText}`, 
      DataFetcherErrorType.UNKNOWN_ERROR,
      { statusCode: response.status, response: data }
    );
  }
  
  // レスポンスデータの妥当性検証
  if (!data || typeof data !== "object") {
    return new DataFetcherError(
      "不正なレスポンス形式：JSONオブジェクトではありません", 
      DataFetcherErrorType.RESPONSE_FORMAT_ERROR,
      { response: data }
    );
  }
  
  // Items配列が存在するか確認
  if (!Array.isArray(data.Items)) {
    return new DataFetcherError(
      "レスポンスに商品データが含まれていません", 
      DataFetcherErrorType.RESPONSE_FORMAT_ERROR,
      { response: data }
    );
  }
  
  return null; // エラーなし
}

// ネットワークエラーをハンドリング
function handleNetworkError(error: unknown): DataFetcherError {
  if (error instanceof Error) {
    const message = error.message.toLowerCase();
    
    if (message.includes("timeout") || message.includes("timed out")) {
      return new DataFetcherError(
        `リクエストタイムアウト: ${error.message}`,
        DataFetcherErrorType.TIMEOUT_ERROR,
        { retryable: true, cause: error }
      );
    }
    
    if (message.includes("connection") || message.includes("network")) {
      return new DataFetcherError(
        `接続エラー: ${error.message}`,
        DataFetcherErrorType.CONNECTION_ERROR,
        { retryable: true, cause: error }
      );
    }
    
    // その他のネットワークエラー
    return new DataFetcherError(
      `ネットワークエラー: ${error.message}`,
      DataFetcherErrorType.NETWORK_ERROR,
      { retryable: true, cause: error }
    );
  }
  
  // 不明なエラー
  return new DataFetcherError(
    `不明なエラー: ${String(error)}`,
    DataFetcherErrorType.UNKNOWN_ERROR,
    { retryable: false }
  );
}
```

## 4. エラー処理戦略

### 4.1 リトライ戦略

リトライ可能なエラーに対して、以下の戦略でリトライを実行します：

```typescript
interface RetryOptions {
  maxRetries: number;      // 最大リトライ回数
  initialDelay: number;    // 初回リトライ前の待機時間（ミリ秒）
  maxDelay: number;        // 最大待機時間（ミリ秒）
  backoffFactor: number;   // バックオフ係数（待機時間の増加率）
  jitter: boolean;         // ジッター（ランダム要素）の有無
}

const defaultRetryOptions: RetryOptions = {
  maxRetries: 3,
  initialDelay: 1000, // 1秒
  maxDelay: 30000,    // 30秒
  backoffFactor: 2,   // 指数バックオフ（2倍ずつ増加）
  jitter: true,       // ジッター有効
};

// リトライ処理を行う関数
async function withRetry<T>(
  operation: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const retryOptions: RetryOptions = {
    ...defaultRetryOptions,
    ...options
  };
  
  let lastError: DataFetcherError | Error;
  
  for (let attempt = 0; attempt <= retryOptions.maxRetries; attempt++) {
    try {
      // 操作を実行
      return await operation();
    } catch (error) {
      // DataFetcherErrorに変換
      const dataFetcherError = error instanceof DataFetcherError 
        ? error 
        : handleNetworkError(error);
      
      lastError = dataFetcherError;
      
      // リトライ可能かつ最大回数に達していない場合はリトライ
      const isLastAttempt = attempt >= retryOptions.maxRetries;
      if (!dataFetcherError.retryable || isLastAttempt) {
        break;
      }
      
      // 待機時間の計算
      let delay = retryOptions.initialDelay * Math.pow(retryOptions.backoffFactor, attempt);
      delay = Math.min(delay, retryOptions.maxDelay);
      
      // ジッターの追加（±10%）
      if (retryOptions.jitter) {
        const jitterFactor = 0.1; // 10%
        const jitterAmount = delay * jitterFactor;
        delay += Math.random() * jitterAmount * 2 - jitterAmount;
      }
      
      // ログ出力
      console.log(
        `リトライ ${attempt + 1}/${retryOptions.maxRetries} ` + 
        `(${Math.round(delay)}ms後): ${dataFetcherError.message}`
      );
      
      // 待機
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  
  // 最終的にエラーとなった場合
  throw lastError;
}
```

### 4.2 致命的エラーと非致命的エラーの区別

エラーを致命的（処理継続不可）と非致命的（一部データ欠損で処理継続）に区分して対応します：

```typescript
// エラーの重大度を判定する関数
function isFatalError(error: DataFetcherError): boolean {
  // 認証エラーは致命的
  if (
    error.type === DataFetcherErrorType.AUTHENTICATION_ERROR || 
    error.type === DataFetcherErrorType.INVALID_API_KEY
  ) {
    return true;
  }
  
  // パラメータエラーは致命的
  if (
    error.type === DataFetcherErrorType.PARAMETER_ERROR ||
    error.type === DataFetcherErrorType.INVALID_GENRE_ID
  ) {
    return true;
  }
  
  // レスポンス形式エラーは致命的
  if (error.type === DataFetcherErrorType.RESPONSE_FORMAT_ERROR) {
    return true;
  }
  
  // 一定回数のリトライ失敗後も発生するネットワークエラーは致命的
  if (
    (error.type === DataFetcherErrorType.NETWORK_ERROR ||
     error.type === DataFetcherErrorType.CONNECTION_ERROR ||
     error.type === DataFetcherErrorType.TIMEOUT_ERROR) &&
    error.cause instanceof Error &&
    error.cause.message.includes("after max retries")
  ) {
    return true;
  }
  
  // その他のエラーは非致命的
  return false;
}
```

### 4.3 部分的な成功の処理

大量データ取得時には、一部のデータ取得に失敗しても全体の処理を継続できるよう、部分的な成功を許容します：

```typescript
interface FetchResult<T> {
  data: T[];                   // 正常に取得できたデータ
  errors: DataFetcherError[];  // 発生したエラー一覧
  totalRequested: number;      // 要求した総データ数
  success: boolean;            // 全体として成功したか
  successRate: number;         // 成功率（0.0～1.0）
}

// 複数ページデータ取得時の部分的成功処理
async function fetchBatchWithPartialSuccess(
  fetchFunctions: Array<() => Promise<any[]>>,
  options: {
    minSuccessRate?: number,           // 最小成功率
    maxConsecutiveErrors?: number,     // 連続エラー最大許容数
    stopOnFatalError?: boolean         // 致命的エラーで全体中断
  } = {}
): Promise<FetchResult<any>> {
  const {
    minSuccessRate = 0.8,           // デフォルト80%以上の成功率を要求
    maxConsecutiveErrors = 3,       // デフォルト連続3回のエラーで中断
    stopOnFatalError = true         // デフォルト致命的エラーで中断
  } = options;
  
  const result: FetchResult<any> = {
    data: [],
    errors: [],
    totalRequested: fetchFunctions.length,
    success: true,
    successRate: 1.0
  };
  
  let consecutiveErrors = 0;
  
  for (const fetchFn of fetchFunctions) {
    try {
      // リトライロジックを含めた実行
      const batchData = await withRetry(fetchFn);
      result.data.push(...batchData);
      consecutiveErrors = 0; // 連続エラーカウントをリセット
    } catch (error) {
      const dataFetcherError = error instanceof DataFetcherError 
        ? error 
        : handleNetworkError(error);
      
      result.errors.push(dataFetcherError);
      consecutiveErrors++;
      
      // 致命的エラーで中断
      if (stopOnFatalError && isFatalError(dataFetcherError)) {
        result.success = false;
        break;
      }
      
      // 連続エラーの最大数を超えた場合中断
      if (consecutiveErrors >= maxConsecutiveErrors) {
        result.success = false;
        break;
      }
    }
  }
  
  // 成功率の計算
  const successCount = result.totalRequested - result.errors.length;
  result.successRate = successCount / result.totalRequested;
  
  // 最小成功率を下回った場合は全体として失敗とする
  if (result.successRate < minSuccessRate) {
    result.success = false;
  }
  
  return result;
}
```

## 5. エラーログとモニタリング

### 5.1 エラーログ出力

エラー情報を適切にログ出力し、問題の追跡と分析を容易にします：

```typescript
enum LogLevel {
  DEBUG = "DEBUG",
  INFO = "INFO",
  WARN = "WARN",
  ERROR = "ERROR",
  FATAL = "FATAL"
}

interface LogEntry {
  timestamp: string;
  level: LogLevel;
  message: string;
  error?: {
    type: string;
    message: string;
    statusCode?: number;
    stack?: string;
  };
  context?: Record<string, any>;
}

// エラーログ出力関数
function logError(
  error: DataFetcherError, 
  context: Record<string, any> = {},
  level: LogLevel = LogLevel.ERROR
): void {
  const logEntry: LogEntry = {
    timestamp: new Date().toISOString(),
    level,
    message: error.message,
    error: {
      type: error.type,
      message: error.message,
      statusCode: error.statusCode,
      stack: error.stack
    },
    context
  };
  
  // 致命的エラーの場合はFATALレベルに引き上げ
  if (isFatalError(error) && level !== LogLevel.FATAL) {
    logEntry.level = LogLevel.FATAL;
  }
  
  // JSON形式でログ出力
  console.error(JSON.stringify(logEntry));
  
  // 必要に応じて外部ログサービスへの送信やアラート通知を実装
}
```

### 5.2 エラーメトリクス収集

エラーの傾向分析や監視のためのメトリクス収集を実装します：

```typescript
interface ErrorMetrics {
  totalErrors: number;
  errorsByType: Record<DataFetcherErrorType, number>;
  errorRate: number;
  totalRequests: number;
  timeWindow: {
    start: Date;
    end: Date;
  };
}

class ErrorMetricsCollector {
  private metrics: ErrorMetrics;
  
  constructor() {
    this.resetMetrics();
  }
  
  resetMetrics(): void {
    const now = new Date();
    this.metrics = {
      totalErrors: 0,
      errorsByType: Object.values(DataFetcherErrorType).reduce((acc, type) => {
        acc[type] = 0;
        return acc;
      }, {} as Record<DataFetcherErrorType, number>),
      errorRate: 0,
      totalRequests: 0,
      timeWindow: {
        start: now,
        end: now
      }
    };
  }
  
  recordRequest(): void {
    this.metrics.totalRequests++;
    this.metrics.timeWindow.end = new Date();
    this.updateErrorRate();
  }
  
  recordError(error: DataFetcherError): void {
    this.metrics.totalErrors++;
    this.metrics.errorsByType[error.type]++;
    this.metrics.timeWindow.end = new Date();
    this.updateErrorRate();
  }
  
  private updateErrorRate(): void {
    if (this.metrics.totalRequests > 0) {
      this.metrics.errorRate = this.metrics.totalErrors / this.metrics.totalRequests;
    }
  }
  
  getMetrics(): ErrorMetrics {
    return { ...this.metrics };
  }
  
  getErrorRateForType(type: DataFetcherErrorType): number {
    if (this.metrics.totalRequests === 0) return 0;
    return this.metrics.errorsByType[type] / this.metrics.totalRequests;
  }
}
```

## 6. 実装例

エラー処理を実装したAPIクライアントの例を示します：

```typescript
class RakutenItemRankingClientWithErrorHandling implements RakutenItemRankingClient {
  private readonly baseUrl = "https://app.rakuten.co.jp/services/api/IchibaItem/Ranking/20170628";
  private readonly applicationId: string;
  private readonly affiliateId?: string;
  private readonly metricsCollector: ErrorMetricsCollector;
  
  constructor(applicationId: string, affiliateId?: string) {
    this.applicationId = applicationId;
    this.affiliateId = affiliateId;
    this.metricsCollector = new ErrorMetricsCollector();
  }
  
  // 単一ページ取得メソッド
  async fetchItems(options: FetchOptions): Promise<RakutenItemResponse> {
    const params = new URLSearchParams({
      applicationId: this.applicationId,
      format: "json",
    });
    
    // オプションパラメータの設定
    if (this.affiliateId) params.append("affiliateId", this.affiliateId);
    if (options.genreId) params.append("genreId", options.genreId);
    if (options.gender !== undefined) params.append("gender", options.gender.toString());
    if (options.page !== undefined) params.append("page", options.page.toString());
    if (options.hits !== undefined) params.append("hits", options.hits.toString());
    
    const url = `${this.baseUrl}?${params.toString()}`;
    
    this.metricsCollector.recordRequest();
    
    try {
      // リトライ機能付きでリクエスト実行
      return await withRetry(async () => {
        try {
          const response = await fetch(url, {
            method: "GET",
            headers: { "Accept": "application/json" },
            timeout: 10000 // 10秒タイムアウト
          });
          
          const data = await response.json();
          
          // エラー検出
          const error = detectAPIError(response, data);
          if (error) {
            throw error;
          }
          
          return data as RakutenItemResponse;
        } catch (error) {
          if (error instanceof DataFetcherError) {
            throw error;
          }
          throw handleNetworkError(error);
        }
      });
    } catch (error) {
      const dataFetcherError = error instanceof DataFetcherError
        ? error
        : handleNetworkError(error);
      
      // エラーを記録
      this.metricsCollector.recordError(dataFetcherError);
      logError(dataFetcherError, { url, options });
      
      throw dataFetcherError;
    }
  }
  
  // 複数ページ取得メソッド（部分的成功を許容）
  async fetchAllItems(options: FetchAllOptions): Promise<RakutenItem[]> {
    const { maxItems, concurrency = 3, ...fetchOptions } = options;
    const hits = options.hits || 30;
    
    try {
      // 初回リクエストで総件数を取得
      const firstResponse = await this.fetchItems({ 
        ...fetchOptions, 
        page: 1,
        hits
      });
      
      const items: RakutenItem[] = firstResponse.Items.map(item => item.Item);
      
      // 必要なページ数を計算
      const totalItems = Math.min(firstResponse.count, maxItems);
      const pageCount = Math.ceil(totalItems / hits);
      
      if (pageCount <= 1) {
        return items.slice(0, maxItems);
      }
      
      // 残りのページを取得するための関数を準備
      const remainingPages = Array.from(
        { length: pageCount - 1 },
        (_, i) => i + 2
      );
      
      const fetchFunctions = remainingPages.map(page => {
        return async () => {
          const response = await this.fetchItems({ ...fetchOptions, page, hits });
          return response.Items.map(item => item.Item);
        };
      });
      
      // 部分的な成功を許容してバッチ処理
      const batchResult = await fetchBatchWithPartialSuccess(fetchFunctions, {
        minSuccessRate: 0.8,
        maxConsecutiveErrors: 3,
        stopOnFatalError: true
      });
      
      // 成功したデータを統合
      items.push(...batchResult.data);
      
      // 警告ログ（一部失敗）
      if (batchResult.errors.length > 0) {
        console.warn(
          `${batchResult.errors.length}件のページ取得に失敗しましたが、` +
          `${batchResult.successRate * 100}%のデータを取得できました。`
        );
      }
      
      return items.slice(0, maxItems);
    } catch (error) {
      const dataFetcherError = error instanceof DataFetcherError
        ? error
        : handleNetworkError(error);
      
      logError(dataFetcherError, { options });
      throw dataFetcherError;
    }
  }
  
  // メトリクス取得メソッド
  getErrorMetrics(): ErrorMetrics {
    return this.metricsCollector.getMetrics();
  }
}
```

## 7. まとめ

本ドキュメントでは、データ取得モジュールにおけるエラー処理の詳細な戦略と実装方法を定義しました。主な特徴は次の通りです：

1. **エラー種別の詳細な定義**: 様々なエラー状況を適切に分類し、それぞれに対応できるようにしています。

2. **リトライ戦略**: 指数バックオフとジッターを用いた効果的なリトライ機構により、一時的なエラーからの回復を促進します。

3. **部分的成功の許容**: 大量データ取得時に、一部のエラーでも全体の処理を継続できるようにしています。

4. **ログとメトリクス**: 適切なログ出力とメトリクス収集により、問題の早期発見と分析が可能です。

これらの機能により、データ取得モジュールは安定性と回復力を備え、楽天市場APIからの商品ランキングデータを効率的かつ安全に取得することができます。
