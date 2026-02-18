# 在庫引当指示コントローラー (AllocateIns Controller)

## 概要

在庫の引当指示（アロケーション）に関する操作を行うコントローラーです。出荷オーダー番号の取得、トランザクション情報の取得、ロック追加（単票・総括）、ロック解除など、GI（出庫指示）処理における在庫確保・解放の機能を提供します。

---

## Model クラス一覧

### AllocateIns

| フィールド名 | 型 | JSON キー | 説明 |
|---|---|---|---|
| shpOrderNo | String | shp_order_no | 出荷オーダー番号 |
| saTransactionId | String | sa_transaction_id | 単票トランザクションID |
| commonCargoIdMethod | Integer | common_cargo_id_method | 共通貨物ID管理方法 |

---

## Request クラス一覧

### Post202Single

ロック追加（単票）リクエストボディ。

| フィールド名 | 型 | JSON キー | 説明 |
|---|---|---|---|
| timeDiff | Integer | time_diff | タイムゾーン差分（分） |
| programId | String | program_id | プログラムID |
| saTransactionId | String | sa_transaction_id | 単票トランザクションID |
| saLineId | String | sa_line_id | 単票明細ID |

### Post202Total

ロック追加（総括）リクエストボディ。

| フィールド名 | 型 | JSON キー | 説明 |
|---|---|---|---|
| timeDiff | Integer | time_diff | タイムゾーン差分（分） |
| programId | String | program_id | プログラムID |
| tpTransactionId | String | tp_transaction_id | 総括トランザクションID |
| tpLineId | String | tp_line_id | 総括明細ID |

---

## Response クラス一覧

### AllocateInsResponse

在庫引当指示の一覧取得レスポンス（ページネーション付き）。

| フィールド名 | 型 | 説明 |
|---|---|---|
| totalPages | Integer | 総ページ数 |
| totalElements | Integer | 総要素数 |
| sort | Sort | ソート情報 |
| first | boolean | 最初のページかどうか |
| last | boolean | 最後のページかどうか |
| pageable | Pageable | ページング情報 |
| content | List\<AllocateIns\> | 引当指示データリスト |
| size | Integer | 1ページあたりの件数 |
| number | Integer | 現在のページ番号 |
| empty | boolean | コンテンツが空かどうか |

### AllocateIns202Get

ロック追加後のレスポンス。

| フィールド名 | 型 | JSON キー | 説明 |
|---|---|---|---|
| lockId | ArrayList\<Integer\> | lock_ids | 追加されたロックIDのリスト |

### AllocateInsTransactionNo

トランザクション番号取得レスポンス。

| フィールド名 | 型 | JSON キー | 説明 |
|---|---|---|---|
| saTransactionId | String | sa_transaction_id | 単票トランザクションID |
| companyCd | String | company_cd | 会社コード |
| customerCd | String | customer_cd | 顧客コード |
| categoryCd | String | category_cd | カテゴリコード |

### AllocationUnlock202A

ロック解除リクエストボディ（DELETEメソッドのボディとして使用）。

| フィールド名 | 型 | JSON キー | 説明 |
|---|---|---|---|
| programId | String | program_id | プログラムID |
| timeDiff | Integer | time_diff | タイムゾーン差分（分） |

### UnlockResponse202Get

ロック解除結果レスポンス。

| フィールド名 | 型 | JSON キー | 説明 |
|---|---|---|---|
| locked | boolean | locked | ロック状態（trueの場合ロック中） |

### UnlockResponseGet

ロック解除結果レスポンス（別形式）。

| フィールド名 | 型 | JSON キー | 説明 |
|---|---|---|---|
| lockIds | boolean | LockIds | ロックIDの存在有無 |

---

## Service クラス

### AllocateInsService (interface)

Retrofit インターフェース。`Observable<Response<T>>` を返す RxJava ベースの非同期APIクライアント。

| メソッド名 | HTTPメソッド | 戻り値 | 説明 |
|---|---|---|---|
| getOrderNumbers | GET | `Observable<Response<BaseResponse<AllocateInsResponse>>>` | 出荷オーダー番号の一覧を取得する。クエリパラメータ: company_cd, warehouse_cd, text_search, ship_date, page, limit |
| getTransactions | GET | `Observable<Response<BaseResponseArray<AllocateInsTransactionNo>>>` | トランザクション番号の一覧を取得する。クエリパラメータ: company_cd, warehouse_cd, shp_order_no, text_search, page, limit |
| postAllocateIns202Single | POST | `Observable<Response<BaseResponse<AllocateIns202Get>>>` | 単票のロックを追加する（GI017 単票ロック追加API） |
| postAllocateIns202Total | POST | `Observable<Response<BaseResponse<AllocateIns202Get>>>` | 総括のロックを追加する（GI017 総括ロック追加API） |

### UnlockRecordService (interface)

ロック解除専用の Retrofit インターフェース。

| メソッド名 | HTTPメソッド | 戻り値 | 説明 |
|---|---|---|---|
| deleteUnlockRecordStocks | DELETE | `Observable<Response<BaseResponse<PickingUnlockRecordGet>>>` | 在庫ロックを解除する。クエリパラメータ: lock_ids（ArrayList\<Integer\>）、ボディ: AllocationUnlock202A |
| deleteUnlockRecordInrQTY | DELETE | `Observable<Response<BaseResponse<PickUnlockRecordInrQTYGet>>>` | 内数数量のロックを解除する。クエリパラメータ: lock_ids（ArrayList\<Integer\>）、ボディ: AllocationUnlock202A |

---

## 使用例

```java
// AllocateInsService の使用例（出荷オーダー番号一覧を取得する）
AllocateInsService service = retrofit.create(AllocateInsService.class);

service.getOrderNumbers(
        "COMP001",   // company_cd
        "WH001",     // warehouse_cd
        null,        // text_search
        null,        // ship_date
        0,           // page
        20           // limit
)
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(response -> {
    if (response.isSuccessful() && response.body() != null) {
        AllocateInsResponse result = response.body().getData();
        List<AllocateIns> list = result.getContent();
        // 一覧を画面に表示する処理
    }
}, throwable -> {
    // エラー処理
});

// Post202Single のロック追加例
Post202Single body = new Post202Single();
body.setProgramId("GI008");
body.setTimeDiff(540); // JST (+9時間 = 540分)
body.setSaTransactionId("SA-0001");
body.setSaLineId("001");

service.postAllocateIns202Single(body)
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(response -> {
    if (response.isSuccessful() && response.body() != null) {
        AllocateIns202Get result = response.body().getData();
        ArrayList<Integer> lockIds = result.getLockId();
        // lockIdsを保持して後でロック解除に使用する
    }
}, throwable -> {
    // エラー処理
});

// UnlockRecordService のロック解除例
UnlockRecordService unlockService = retrofit.create(UnlockRecordService.class);

AllocationUnlock202A unlockBody = new AllocationUnlock202A();
unlockBody.setProgramId("GI008");
unlockBody.setTimeDiff(540);

ArrayList<Integer> lockIds = new ArrayList<>();
lockIds.add(101);
lockIds.add(102);

unlockService.deleteUnlockRecordStocks(lockIds, unlockBody)
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(response -> {
    // ロック解除完了処理
}, throwable -> {
    // エラー処理
});
```
