# QRV (Qualfia Reservation) 業務フロー図

**作成日:** 2025-01-26
**対象システム:** QRV (Qualfia Reservation)
**参照:** QRV_要件定義書.md v1.3

---

## 目次

1. [全体フロー](#1-全体フロー)
2. [予約受付フロー（詳細）](#2-予約受付フロー詳細)
3. [SMS確認フロー](#3-sms確認フロー)
4. [電話代行フロー](#4-電話代行フロー)
5. [決済フロー](#5-決済フロー)
6. [通知フロー](#6-通知フロー)

---

## 1. 全体フロー

システム全体の業務フローを示します。

```mermaid
graph TB
    Start([依頼者がAIとチャット開始]) --> Chat[予約内容を対話で確認]
    Chat --> Confirm{全情報揃った?}
    Confirm -->|No| Chat
    Confirm -->|Yes| CreateReq[予約リクエスト作成]
    CreateReq --> GenID[予約ID発行]
    GenID --> SendSMS1[SMS確認リンク送信]

    SendSMS1 --> WaitSMS{5分以内に<br/>進める押下?}
    WaitSMS -->|キャンセル| Cancel1[予約削除]
    WaitSMS -->|タイムアウト| Cancel1
    WaitSMS -->|進める| CallShops[提携店に電話代行開始]

    CallShops --> CallResult{電話結果}
    CallResult -->|全店NG| Alternative[代替案SMS送信]
    CallResult -->|予約成功| SendPayment[決済リンクSMS送信]

    SendPayment --> WaitPayment{10分以内に<br/>決済完了?}
    WaitPayment -->|No| Release[予約解放通知]
    WaitPayment -->|Yes| Paid[予約確定]

    Paid --> NotifyBoth[依頼者・提携店に確定通知]
    NotifyBoth --> Reminder[予約日24時間前に<br/>リマインダー送信]
    Reminder --> End([予約完了])

    Cancel1 --> End1([終了])
    Alternative --> End1
    Release --> End1

    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style End1 fill:#ffcdd2
    style Paid fill:#fff9c4
```

---

## 2. 予約受付フロー（詳細）

AIエージェントとの対話による予約受付の詳細フローです。

```mermaid
graph TB
    Start([依頼者メッセージ受信]) --> Parse[メッセージ解析]
    Parse --> CheckService{サービスタイプ<br/>特定可能?}

    CheckService -->|No| AskService[サービス一覧を提示<br/>選択を依頼]
    AskService --> WaitService[依頼者の回答待ち]
    WaitService --> Parse

    CheckService -->|Yes| LoadDef[サービス定義ファイル読込]
    LoadDef --> CheckFields[必須項目チェック]

    CheckFields --> AllFields{全項目揃った?}
    AllFields -->|No| AskMissing[不足項目を質問]
    AskMissing --> WaitAnswer[依頼者の回答待ち]
    WaitAnswer --> CheckFields

    AllFields -->|Yes| Validate[入力値検証]
    Validate --> ValidResult{検証結果}

    ValidResult -->|電話番号形式エラー| ErrPhone[電話番号形式エラー表示<br/>再入力依頼]

    ErrPhone --> WaitAnswer

    ValidResult -->|OK| ShowSummary[予約内容サマリ表示]
    ShowSummary --> AskConfirm[進めてよいか確認]
    AskConfirm --> WaitConfirm{依頼者の回答}

    WaitConfirm -->|修正依頼| CheckModify{何を修正?}
    CheckModify --> AskMissing

    WaitConfirm -->|承認| CreateReq[予約リクエスト作成<br/>ステータス: pending_confirmation]
    CreateReq --> GenID[予約ID発行<br/>例: QRV-A3F9D2]
    GenID --> ReturnID[予約IDを表示<br/>SMS送信予告]
    ReturnID --> End([次フローへ])

    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style CreateReq fill:#fff9c4
```

---

## 3. SMS確認フロー

依頼者に確認リンクを送信し、承認を得るフローです。

```mermaid
graph TB
    Start([予約リクエスト作成完了]) --> CreateLink[確認リンク生成<br/>有効期限: 5分]
    CreateLink --> CreateSMS[SMS本文作成<br/>予約内容+進める/キャンセルボタン]
    CreateSMS --> SendSMS[Twilio経由でSMS送信]

    SendSMS --> SMSResult{送信成功?}
    SMSResult -->|No| Retry{リトライ回数<br/>3回未満?}
    Retry -->|Yes| Wait1[1秒待機]
    Wait1 --> SendSMS
    Retry -->|No| NotifyUser[AIチャットに<br/>SMS送信失敗を表示<br/>電話番号確認を促す]

    SMSResult -->|Yes| NotifySuccess[AIチャットに<br/>SMS送信成功を表示]
    NotifySuccess --> UpdateStatus1[ステータス更新<br/>pending_confirmation]
    UpdateStatus1 --> WaitAction[依頼者のアクション待ち<br/>タイマー: 5分]

    WaitAction --> Action{依頼者の<br/>アクション}

    Action -->|進めるボタン| UpdateStatus2[ステータス更新<br/>confirmed]
    Action -->|キャンセルボタン| DeleteReq[予約リクエスト削除]
    Action -->|5分経過| DeleteReq

    UpdateStatus2 --> StartCall[電話代行タスク開始]
    StartCall --> End1([電話代行フローへ])

    DeleteReq --> End2([終了・通知なし])
    NotifyUser --> End3([依頼者対応待ち])

    style Start fill:#e1f5ff
    style End1 fill:#c8e6c9
    style End2 fill:#ffcdd2
    style End3 fill:#fff3e0
```

---

## 4. 電話代行フロー

提携店への自動音声電話による予約取得フローです。

```mermaid
graph TB
    Start([SMS確認完了]) --> UpdateStatus[ステータス更新<br/>calling_shops]
    UpdateStatus --> LoadShops[提携店CSV読込<br/>data/shops_service_type.csv]
    LoadShops --> FilterArea[対応エリアでフィルタ]
    FilterArea --> FilterTime[電話受付時間内でフィルタ<br/>現在時刻が受付時間内の店舗のみ]
    FilterTime --> CheckAvailable{受付時間内の<br/>店舗あり?}

    CheckAvailable -->|No| NotifyOutOfHours[受付時間外SMS送信<br/>「現在受付時間外のため<br/>予約できませんでした」]
    NotifyOutOfHours --> UpdateFailed2[ステータス更新<br/>failed]
    UpdateFailed2 --> End3([受付時間外終了])

    CheckAvailable -->|Yes| SortScore[スコア降順でソート]
    SortScore --> InitRound[試行回数 = 1周目]

    InitRound --> CheckList{提携店リスト<br/>残あり?}
    CheckList -->|No| CheckRound{試行回数}
    CheckList -->|Yes| NextShop[次の提携店を取得]

    NextShop --> CallShop[Bland.ai経由で<br/>自動音声電話]
    CallShop --> Opening[Opening<br/>挨拶・担当者確認]
    Opening --> Requirements[Requirements<br/>予約内容伝達]
    Requirements --> WaitResponse[店側の応答待ち<br/>タイムアウト: 30秒]

    WaitResponse --> Response{応答内容}

    Response -->|予約OK| Confirmation[Confirmation<br/>予約内容確認]
    Confirmation --> LogSuccess[通話ログ記録<br/>status: success]
    LogSuccess --> UpdateReserved[ステータス更新<br/>reserved<br/>提携店ID記録]
    UpdateReserved --> End1([決済フローへ])

    Response -->|時間帯埋まり| LogBusy[通話ログ記録<br/>status: time_unavailable<br/>代替案があれば記録]
    Response -->|エリア対応不可| LogArea[通話ログ記録<br/>status: area_unavailable]
    Response -->|応答なし| LogNoAnswer[通話ログ記録<br/>status: no_answer]
    Response -->|追加情報要求| LogAddInfo[通話ログ記録<br/>status: additional_info_required]

    LogBusy --> CheckList
    LogArea --> CheckList
    LogNoAnswer --> CheckList
    LogAddInfo --> CheckList

    CheckRound -->|1周目| SecondRound[試行回数 = 2周目<br/>リスト先頭に戻る]
    SecondRound --> CheckList

    CheckRound -->|2周目| AllFailed[全店NG処理]
    AllFailed --> ExtractAlt[通話ログから代替案抽出]
    ExtractAlt --> SendAltSMS[代替案SMS送信]
    SendAltSMS --> UpdateFailed[ステータス更新<br/>failed]
    UpdateFailed --> End2([終了])

    style Start fill:#e1f5ff
    style End1 fill:#c8e6c9
    style End2 fill:#ffcdd2
    style End3 fill:#fff3e0
    style UpdateReserved fill:#fff9c4
```

---

## 5. 決済フロー

仮予約成功後の決済処理フローです。

```mermaid
graph TB
    Start([仮予約成功]) --> CalcPrice[料金計算<br/>サービスタイプ別ロジック]
    CalcPrice --> CreateSession[Stripe Checkout Session作成<br/>有効期限: 10分]
    CreateSession --> UpdateStatus1[ステータス更新<br/>payment_pending]

    UpdateStatus1 --> CreateSMS1[依頼者向けSMS作成<br/>予約内容+決済リンク]
    CreateSMS1 --> SendSMS1[Twilio経由でSMS送信]
    SendSMS1 --> SMS1Result{送信成功?}

    SMS1Result -->|No| Retry1{リトライ<br/>3回未満?}
    Retry1 -->|Yes| Wait1[1秒待機]
    Wait1 --> SendSMS1
    Retry1 -->|No| AutoCancel[予約自動キャンセル<br/>提携店にも通知]
    AutoCancel --> End3([終了])

    SMS1Result -->|Yes| CreateSMS2[提携店向けSMS作成<br/>予約内容+決済待ち通知]
    CreateSMS2 --> SendSMS2[SMS/Email送信]

    SendSMS2 --> StartTimer[決済タイマー開始<br/>10分]
    StartTimer --> WaitPayment{依頼者の<br/>アクション}

    WaitPayment -->|決済リンククリック| ShowCheckout[Stripe Checkout表示]
    ShowCheckout --> PayAction{決済操作}

    PayAction -->|決済完了| Webhook[Stripe Webhook受信]
    Webhook --> UpdatePaid[ステータス更新<br/>paid<br/>決済情報記録]
    UpdatePaid --> SendConfirmSMS1[依頼者に決済完了SMS]
    SendConfirmSMS1 --> SendConfirmSMS2[提携店に決済完了SMS/Email]
    SendConfirmSMS2 --> End1([予約確定完了])

    PayAction -->|決済失敗| WaitPayment
    WaitPayment -->|10分経過| ReleaseReq[予約解放]
    ReleaseReq --> SendReleaseSMS1[依頼者に解放通知SMS]
    SendReleaseSMS1 --> SendReleaseSMS2[提携店に解放通知SMS]
    SendReleaseSMS2 --> End2([終了])

    style Start fill:#e1f5ff
    style End1 fill:#c8e6c9
    style End2 fill:#ffcdd2
    style End3 fill:#ffcdd2
    style UpdatePaid fill:#fff9c4
```

---

## 6. 通知フロー

リマインダーおよび各種通知の送信フローです。

```mermaid
graph TB
    Start([Celery Beat定期実行<br/>毎時0分]) --> Query[DB検索<br/>予約日時の24時間前]
    Query --> HasTarget{対象予約あり?}

    HasTarget -->|No| End1([終了])
    HasTarget -->|Yes| ForEach[各予約に対して処理]

    ForEach --> CreateSMS1[依頼者向け<br/>リマインダーSMS作成]
    CreateSMS1 --> SendSMS1[Twilio経由で送信]
    SendSMS1 --> SMS1Result{送信成功?}

    SMS1Result -->|No| Retry1{リトライ<br/>3回未満?}
    Retry1 -->|Yes| Wait1[1秒待機]
    Wait1 --> SendSMS1
    Retry1 -->|No| LogError1[エラーログ記録<br/>処理は継続]

    SMS1Result -->|Yes| LogSuccess1[送信ログ記録]
    LogSuccess1 --> CreateSMS2[提携店向け<br/>リマインダーSMS作成]
    LogError1 --> CreateSMS2

    CreateSMS2 --> SendSMS2[SMS送信]
    SendSMS2 --> SMS2Result{送信成功?}

    SMS2Result -->|No| Retry2{リトライ<br/>3回未満?}
    Retry2 -->|Yes| Wait2[1秒待機]
    Wait2 --> SendSMS2
    Retry2 -->|No| LogError2[エラーログ記録]

    SMS2Result -->|Yes| LogSuccess2[送信ログ記録]

    LogSuccess2 --> SendEmail[提携店向け<br/>リマインダーEmail送信]
    LogError2 --> SendEmail

    SendEmail --> EmailResult{送信成功?}
    EmailResult -->|No| LogError3[エラーログ記録]
    EmailResult -->|Yes| LogSuccess3[送信ログ記録]

    LogSuccess3 --> NextReservation{次の予約あり?}
    LogError3 --> NextReservation

    NextReservation -->|Yes| ForEach
    NextReservation -->|No| End2([全件処理完了])

    style Start fill:#e1f5ff
    style End1 fill:#e0e0e0
    style End2 fill:#c8e6c9
```

---

## 7. ステータス遷移図

予約のステータス遷移を示します。

```mermaid
stateDiagram-v2
    [*] --> pending_confirmation: 予約リクエスト作成

    pending_confirmation --> confirmed: SMS確認完了(進める)
    pending_confirmation --> cancelled: SMS確認(キャンセル)
    pending_confirmation --> cancelled: 5分タイムアウト

    confirmed --> calling_shops: 電話代行開始

    calling_shops --> reserved: いずれかの店で予約成功
    calling_shops --> failed: 全店NG(2周完了)

    reserved --> payment_pending: 決済リンク送信

    payment_pending --> paid: 決済完了
    payment_pending --> cancelled: 10分タイムアウト

    paid --> [*]: 予約完了
    cancelled --> [*]
    failed --> [*]
```

---

## 8. エラー処理フロー

主要なエラーケースの処理フローです。

```mermaid
graph TB
    Start([エラー発生]) --> ErrorType{エラー種別}

    ErrorType -->|SMS送信失敗| SMSError[SMS送信エラー処理]
    ErrorType -->|電話接続失敗| CallError[電話エラー処理]
    ErrorType -->|決済失敗| PaymentError[決済エラー処理]
    ErrorType -->|システムエラー| SystemError[システムエラー処理]

    SMSError --> SMSRetry{リトライ回数<br/>3回未満?}
    SMSRetry -->|Yes| SMSWait[1秒待機]
    SMSWait --> SMSResend[再送信]
    SMSResend --> SMSResult{成功?}
    SMSResult -->|Yes| SMSLog[成功ログ記録]
    SMSResult -->|No| SMSRetry

    SMSRetry -->|No| SMSFinalFail[最終失敗処理]
    SMSFinalFail --> SMSNotify[依頼者に代替通知<br/>またはキャンセル]

    CallError --> CallLog[通話ログ記録<br/>status: failed]
    CallLog --> CallNext[次の提携店へ]

    PaymentError --> PaymentLog[エラーログ記録]
    PaymentLog --> PaymentNotify[依頼者に再試行促す]

    SystemError --> AdminNotify[管理者にSMS通知<br/>080-5496-0516]
    AdminNotify --> SystemLog[詳細エラーログ記録]
    SystemLog --> SystemRecover{自動復旧可能?}

    SystemRecover -->|Yes| Retry[処理再試行]
    SystemRecover -->|No| ManualAction[管理者対応待ち]

    SMSLog --> End1([処理継続])
    SMSNotify --> End2([エラー終了])
    CallNext --> End1
    PaymentNotify --> End1
    Retry --> End1
    ManualAction --> End3([管理者対応待ち])

    style Start fill:#ffcdd2
    style End1 fill:#c8e6c9
    style End2 fill:#ffcdd2
    style End3 fill:#fff3e0
```

---

## 9. サービスタイプ別処理分岐

サービスタイプに応じた処理の分岐を示します。

```mermaid
graph TB
    Start([予約リクエスト受信]) --> LoadType[service_type取得]
    LoadType --> Branch{service_type}

    Branch -->|00001| Aroma[アロママッサージ]
    Branch -->|00002| Thai[タイマッサージ]
    Branch -->|00003| Orchid[胡蝶蘭配送]
    Branch -->|その他| Unknown[未対応サービス]

    Aroma --> LoadAromaDef[config/services/00001.yaml読込]
    Thai --> LoadThaiDef[config/services/00002.yaml読込]
    Orchid --> LoadOrchidDef[config/services/00003.yaml読込]

    LoadAromaDef --> ValidateAroma[アロマ固有検証<br/>- 時間: 40-120分<br/>- エリア: 東京23区]
    LoadThaiDef --> ValidateThai[タイ固有検証<br/>- 時間: 60-120分<br/>- エリア: 東京23区]
    LoadOrchidDef --> ValidateOrchid[胡蝶蘭固有検証<br/>- サイズ: 3/6/10本立て<br/>- エリア: 東京23区]

    ValidateAroma --> CalcAromaPrice[料金計算<br/>時間帯考慮<br/>6:00-22:00: 1,200円/10分<br/>22:00-6:00: 1,500円/10分]
    ValidateThai --> CalcThaiPrice[料金計算<br/>時間帯考慮<br/>6:00-22:00: 1,300円/10分<br/>22:00-6:00: 1,600円/10分]
    ValidateOrchid --> CalcOrchidPrice[料金計算<br/>サイズ固定<br/>3本: 15,000円<br/>6本: 25,000円<br/>10本: 30,000円]

    CalcAromaPrice --> LoadAromaShops[data/shops_00001.csv読込]
    CalcThaiPrice --> LoadThaiShops[data/shops_00002.csv読込]
    CalcOrchidPrice --> LoadOrchidShops[data/shops_00003.csv読込]

    LoadAromaShops --> Common[共通処理へ]
    LoadThaiShops --> Common
    LoadOrchidShops --> Common

    Unknown --> ShowError[エラー表示<br/>対応サービス一覧提示]
    ShowError --> End2([終了])

    Common --> End1([次フローへ])

    style Start fill:#e1f5ff
    style End1 fill:#c8e6c9
    style End2 fill:#ffcdd2
```

---

**文書終了**
