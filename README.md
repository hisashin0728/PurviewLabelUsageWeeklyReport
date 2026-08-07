# Purview Label Usage Weekly Report — Security Copilot カスタムエージェント

**DLP ポリシー（ルール）を有効化していない環境**（秘密度ラベルの付与のみを行う環境）向けに、
Microsoft Purview の**秘密度ラベルの利用状況**を、Microsoft Defender の Advanced Hunting テーブル
**`CloudAppEvents`** から KQL で**週次**集計し、視認性の高い **HTML/CSS レポート**を
**Azure Logic App 経由でメール配信**する Security Copilot カスタムエージェントです。

ラベルの **付与 / 変更 / 削除** を、Purview の統合監査ログ（`CloudAppEvents`）を基に集計し、
**適用された秘密度ラベル（親 / 子）**、**アクション別の内訳**、**アプリ／ワークロード別**、
**ユーザー別の偏り**、**格下げ・削除の注視**、**対象ファイルの種別・パス**、
**SharePoint サイト別の機密系ラベル付与状況**を判別します。

> 姉妹エージェント [PurviewDlpWeeklyReport](../PurviewDlpWeeklyReport/README.md) は DLP アラート
> （`DLPRuleMatch`）を対象としますが、本エージェントは **DLP ルール不要**で、ラベル監査イベント
> （`FileSensitivityLabelApplied` 等）を対象とします。

---

## なぜ CloudAppEvents のラベル監査イベントなのか

DLP ポリシー（ルール）を有効化していない環境では、`DLPRuleMatch` レコードは発生しません。
そのため DLP ベースの集計は 0 件になります。一方、秘密度ラベルの**付与・変更・削除**は、
DLP の有無に関係なく**統合監査ログ**に記録され、`CloudAppEvents` から取得できます。

- **DLP ルール不要**（ラベル付与のみの環境で動作）
- M365 統合監査ログ全体を **1 テーブル**で KQL 集計可能（SharePoint / OneDrive / Exchange /
  Office クライアント / Teams 横断）
- Defender Advanced Hunting で自動化・週次レポート化できる

### 扱う対象・扱わない対象

| 項目 | 本エージェント | 補足 |
| --- | --- | --- |
| ラベルの付与 / 変更 / 削除 | ○ | 主対象 |
| ラベル分布・カバレッジ（利用の広がり） | ○ | ユーザー数 / ファイル数 |
| 格下げ / 格上げ / 削除の注視 | ○ | 旧→新ラベルの優先度比較（ヒューリスティック） |
| アプリ／ワークロード別・ユーザー別 | ○ | どこで・誰が |
| SharePoint サイト別のラベル付与 | ○ | 機密系ラベルが多いサイトを把握 |
| DLP 違反・機密情報の種類（SIT） | ✕ | DLP ルールが必要（PurviewDlpWeeklyReport を参照） |
| ブロック / 検知の判定・持ち出し経路 | ✕ | DLP ルールが必要 |
| 現存する全ファイルのラベル棚卸し | ✕ | イベント履歴のため。Purview Content Explorer / Information Protection Reports を参照 |

---

## 使用テーブルと対象アクティビティ

### `CloudAppEvents`（Defender Advanced Hunting）

`CloudAppEvents` は Microsoft 365 の統合監査ログ（秘密度ラベル アクティビティを含む）を保持します。
ラベルの詳細は `RawEventData`（元の監査イベント JSON）内のフィールドに格納されます。

### 対象アクティビティ（`RawEventData.Operation`）

| アクション | Operation |
| --- | --- |
| **付与** | `FileSensitivityLabelApplied` / `SensitivityLabelApplied` / `SiteSensitivityLabelApplied` |
| **変更** | `FileSensitivityLabelChanged` / `SensitivityLabelUpdated` / `SensitivityLabelChanged` / `SiteSensitivityLabelChanged` |
| **削除** | `FileSensitivityLabelRemoved` / `SensitivityLabelRemoved` / `SiteSensitivityLabelRemoved` |

### 使用する主なフィールド

| フィールド | 用途 |
| --- | --- |
| `Timestamp` | 対象期間フィルタ（週次 / 前週比） |
| `RawEventData.Operation` | アクション分類（付与 / 変更 / 削除） |
| `RawEventData.SensitivityLabelEventData.SensitivityLabelId` / `RawEventData.SensitivityLabelId` / `RawEventData.LabelId` | **適用ラベル（GUID）**。`LabelMap` で名称に変換 |
| `RawEventData.SensitivityLabelEventData.OldSensitivityLabelId` | **旧ラベル（GUID）**。格下げ / 格上げ判定に使用 |
| `RawEventData.SensitivityLabelEventData.JustificationText` | 変更・格下げの**理由テキスト**（あれば） |
| `RawEventData.Workload` / `Application` | アプリ／ワークロード（Word / Excel / SharePoint / Exchange 等） |
| `AccountDisplayName` / `RawEventData.UserId` / `AccountId` | ユーザー（実操作者） |
| `RawEventData.SourceFileName` / `ObjectName` / `ObjectId` / `SourceRelativeUrl` | 対象ファイル名・パス |
| `RawEventData.SiteUrl` / `ObjectId` | SharePoint サイト URL の特定 |

---

## KQL スキル一覧

すべて **Target: Defender**（Advanced Hunting）で `CloudAppEvents` を対象とし、過去 14 日間
（今週 / 前週の比較）を集計します。

| スキル | 目的 | レポート項目 |
| --- | --- | --- |
| `GetLabelActionSummary` | 付与/変更/削除の固定 3 行で今週・前週・増減・14日間合計を集計 | 1. 対象期間 / 2. サマリー / 3-2. アクション別集計 |
| `GetLabelActivitySummary` | 週 × アクション × ワークロード × ラベル別の件数 | 3-3. アプリ／ワークロード別 |
| `GetLabelDistribution` | ラベル別 × 週の付与/変更/削除、ユニークユーザー/ファイル数 | 3-1. ラベル分布 |
| `GetLabelActivityByUser` | ユーザー別の付与/変更/削除（上位 8 件） | 3-4. ユーザー分析 |
| `GetLabelDowngradeRemoval` | 格下げ/格上げ/削除（旧→新ラベル、理由） | 3-5. 格下げ・削除の注視 |
| `GetLabelFileStatistics` | ファイル種別 × ラベル × 週の件数とパス | 3-6. ファイル分析 |
| `GetLabelBySharePointSite` | SharePoint サイト別の付与数・機密系付与数・ユーザー数 | 3-7. SharePoint サイト別分析 |
| `SendLabelReportEmail` | HTML レポートのメール送信（LogicApp） | 配信 |

`GetLabelActionSummary` は総アクティビティ数と付与・変更・削除件数の唯一の集計元です。
今週・前週とも `総数 = 付与 + 変更 + 削除` を検算し、他スキルの結果から再集計しません。

> 生成レポートのイメージは同梱の
> [PurviewLabelUsageWeeklyReport_SampleEmail.html](PurviewLabelUsageWeeklyReport_SampleEmail.html)
> （ダミーデータ）を参照してください。

---

## 秘密度ラベルは GUID で記録されます（重要）

`CloudAppEvents` の `RawEventData` では、秘密度ラベルは **GUID（`SensitivityLabelId`）** で
格納されます。これは組み込みラベルもカスタムラベルも同じです。

- **組み込み既定ラベル**は固定 GUID（`defa4170-0d19-0005-000X-...`）のため、各スキルの `LabelMap`
  にハードコード済みで名称解決されます。
- **カスタムラベル**はテナント固有のランダム GUID のため、`LabelMap` に登録しない限り
  **GUID のまま表示**されます。
- **親ラベル / 子ラベル（サブラベル）**: ログに記録されるのは適用された**子ラベルの GUID のみ**で、
  親ラベルの GUID や階層は含まれません。`LabelMap` に親ラベル名（`ParentName`）を登録すると、
  ラベルを「**親 / 子**」（例: `Confidential / All Employees`）で表示できます。

### 【お客様作業】カスタムラベルの GUID とラベル名を抽出するスクリプト

以下の PowerShell を実行し、テナントの全ラベルの **GUID（ImmutableId）／子ラベル名／親ラベル名／
優先度（Priority）** を、各 KQL スキルの `LabelMap` datatable にそのまま貼り付けられる形式で出力します。
Security & Compliance PowerShell（`Connect-IPPSSession`）を使用します。

```powershell
# 0) 事前準備（未導入の場合のみ）: Exchange Online Management モジュール
Install-Module ExchangeOnlineManagement -Scope CurrentUser

# 1) Security & Compliance PowerShell に接続（Compliance 管理者などの権限が必要）
Connect-IPPSSession -UserPrincipalName admin@contoso.onmicrosoft.com

# 2) テナントの全ラベルを取得
$labels = Get-Label

# 3) 親ラベル名を解決するための ImmutableId -> DisplayName 辞書
$nameById = @{}
foreach ($l in $labels) { $nameById[[string]$l.ImmutableId] = $l.DisplayName }

# 4) KQL datatable 行を生成: "GUID","子ラベル名","親ラベル名",Priority
#    ※ $pid は PowerShell の自動変数（読み取り専用）のため使用しない。$parentId を使う。
$rows = foreach ($l in $labels) {
    $guid     = [string]$l.ImmutableId
    $child    = ($l.DisplayName) -replace '"','""'
    $parentId = [string]$l.ParentId
    $parent = if ($parentId -and $parentId -ne '00000000-0000-0000-0000-000000000000' -and $nameById.ContainsKey($parentId)) {
                  $nameById[$parentId]
              } else { '' }
    $parent = $parent -replace '"','""'
    $prio   = if ($null -ne $l.Priority) { [int]$l.Priority } else { 0 }
    '              "{0}","{1}","{2}",{3},' -f $guid, $child, $parent, $prio
}

# 5) 末尾カンマを除去して出力（クリップボードにもコピー）
$block = ($rows -join "`n").TrimEnd(',')
$block | Set-Clipboard
Write-Host "=== 以下を各 KQL スキルの LabelMap datatable の行に貼り付けてください ===`n"
$block

# （参考）人が確認しやすい一覧表示
$labels | Select-Object `
    @{n='GUID(ImmutableId)';e={[string]$_.ImmutableId}}, `
    DisplayName, `
    @{n='ParentName';e={ $p=[string]$_.ParentId; if ($nameById.ContainsKey($p)) { $nameById[$p] } else { '' } }}, `
    Priority |
    Sort-Object Priority | Format-Table -AutoSize
```

> **出力例（貼り付け用）:**
> ```kusto
>               "8f2a1c3d-...-child1","All Employees","Confidential",5,
>               "9b3c2d4e-...-child2","Anyone","Confidential",6,
>               "a1b2c3d4-...-hc-all","All Employees","Highly Confidential",9
> ```
> - `GUID(ImmutableId)` が、`CloudAppEvents` の `SensitivityLabelId` と一致する値です。
> - 親を持たないラベルは `ParentName` が空文字（`""`）になります。
> - `Priority` は格下げ / 格上げ判定に使用します（値が大きいほど高機密）。

### LabelMap への反映手順

1. 上記スクリプトの出力（`$block`）をコピーします。
2. [PurviewLabelUsageWeeklyReport.yaml](PurviewLabelUsageWeeklyReport.yaml) を開き、**6 つの KQL スキル**
   （`GetLabelActivitySummary` / `GetLabelDistribution` / `GetLabelActivityByUser` /
  `GetLabelDowngradeRemoval` / `GetLabelFileStatistics` / `GetLabelBySharePointSite`）の各 `LabelMap`
  datatable にある
   `// カスタム/子ラベルは …` のコメント行を、コピーした行で置き換え（または追記）します。
3. datatable の**最終行にはカンマを付けない**でください（`];` の直前）。
4. 組み込みラベルの行はそのまま残して構いません（重複する GUID があればカスタム側を優先し、重複行は削除）。

---

## 【動作確認】お客様環境で本エージェントが動くかのテスト KQL

本エージェントを配置する前に、Microsoft Defender ポータルの **Advanced hunting** で以下を実行し、
`CloudAppEvents` にラベル監査イベントが届いているかを確認してください。

### テスト 0: テーブルの存在確認

```kusto
CloudAppEvents
| take 1
```

- **結果が返る** → `CloudAppEvents` が利用可能（Defender for Cloud Apps 接続済み）。
- **`cannot resolve table 'CloudAppEvents'` エラー** → Defender for Cloud Apps が未展開。
  先に Microsoft 365 を Defender for Cloud Apps に接続してください。

### テスト 1: ラベル監査イベントの有無（レディネスチェック）

```kusto
let labelOps = dynamic([
  "FileSensitivityLabelApplied","SensitivityLabelApplied","SiteSensitivityLabelApplied",
  "FileSensitivityLabelChanged","SensitivityLabelUpdated","SensitivityLabelChanged","SiteSensitivityLabelChanged",
  "FileSensitivityLabelRemoved","SensitivityLabelRemoved","SiteSensitivityLabelRemoved"
]);
CloudAppEvents
| where Timestamp > ago(14d)
| extend Operation = tostring(RawEventData.Operation)
| where Operation in (labelOps)
| summarize Events = count() by Operation
| sort by Events desc
```

### テスト 2: エージェントが使うフィールドの中身を確認

```kusto
let labelOps = dynamic([
  "FileSensitivityLabelApplied","SensitivityLabelApplied","SiteSensitivityLabelApplied",
  "FileSensitivityLabelChanged","SensitivityLabelUpdated","SensitivityLabelChanged","SiteSensitivityLabelChanged",
  "FileSensitivityLabelRemoved","SensitivityLabelRemoved","SiteSensitivityLabelRemoved"
]);
CloudAppEvents
| where Timestamp > ago(14d)
| extend Operation = tostring(RawEventData.Operation)
| where Operation in (labelOps)
| extend SLE = parse_json(tostring(RawEventData.SensitivityLabelEventData))
| extend LabelGUID = coalesce(tostring(SLE.SensitivityLabelId), tostring(RawEventData.SensitivityLabelId), tostring(RawEventData.LabelId), "")
| extend OldLabelGUID = coalesce(tostring(SLE.OldSensitivityLabelId), tostring(RawEventData.OldSensitivityLabelId), "")
| extend Workload = coalesce(tostring(RawEventData.Workload), Application, "不明")
| extend User = coalesce(AccountDisplayName, tostring(RawEventData.UserId), AccountId, "不明")
| extend FileName = coalesce(tostring(RawEventData.SourceFileName), tostring(RawEventData.ObjectName), tostring(RawEventData.ObjectId))
| project Timestamp, Operation, Workload, User, LabelGUID, OldLabelGUID, FileName
| take 20
```

### 何が見えていれば本エージェントは動作するか

| テスト 2 の列 | 期待される内容 | この列が見えると… |
| --- | --- | --- |
| `Operation` | `FileSensitivityLabelApplied` 等が **1 件以上** | ラベル監査が `CloudAppEvents` に届いている（**動作の必須条件**）。付与/変更/削除が揃うほど各セクションが充実 |
| `LabelGUID` | GUID 文字列（空でない） | ラベル ID を取得できている。組み込み GUID なら名称解決、カスタム GUID なら `LabelMap` へ追記が必要（3-1・上記スクリプト参照） |
| `Workload` | `Word` / `Excel` / `SharePoint` / `Exchange` 等 | 3-3 アプリ／ワークロード別が生成できる |
| `User` | UPN / 表示名（`APP@…` はシステム自動） | 3-4 ユーザー分析が生成できる |
| `OldLabelGUID` | 変更イベントで GUID が入る | 3-5 の格下げ/格上げ判定ができる（空なら「変更（不明）」に分類） |
| `FileName` | ファイル名・パス | 3-6 ファイル分析が生成できる |

**判定基準:**

- ✅ **テスト 1 が 1 行以上返る** → 本エージェントは実データでレポートを生成できます。
  理想的には `Applied`（付与）を中心に、`Changed`（変更）・`Removed`（削除）も見えていること。
- ⚠️ **テスト 1 が 0 件（テスト 0 は成功）** → テーブルはあるがラベルイベントが届いていません。次を確認:
  1. **統合監査ログ（Unified Audit Log）が有効か**（Purview / Defender の監査がオン）。
  2. **Defender for Cloud Apps のアプリコネクタで「Microsoft 365 activities」が有効か**
     （Defender ポータル > Settings > Cloud apps > App connectors）。接続直後は**取り込みに
     数時間〜最大 24–48 時間**のラグがあります。
  3. **対象期間にラベル操作が発生していない**可能性 → `ago(14d)` を `ago(30d)` に広げて再確認
     （Advanced Hunting の保持期間は約 30 日）。
  4. サイトラベルのみを使用している場合は `Site…` 系 Operation が中心になります。
- ⚠️ **`LabelGUID` は見えるが名称が GUID のまま** → カスタムラベルです。上記 PowerShell で
  抽出し、6 スキルの `LabelMap` に追記すると「親 / 子」で名称表示されます（レポート生成自体は可能）。

### 記録される範囲（ローカルファイル vs クラウド上のファイル）

秘密度ラベルの付与・変更・削除は、**クラウド上のファイル（SharePoint / OneDrive / Teams）だけでなく、
ローカルファイルへの操作も `CloudAppEvents` に記録されます**。記録可否を決めるのは
**「ファイルの保存場所」ではなく「ラベル操作を行ったアプリ」**です。

| ラベル操作を行うアプリ | 記録 | 補足 |
| --- | --- | --- |
| Word / Excel / PowerPoint（**デスクトップ含む**） | ✅ | 保存時にイベント生成。**対象が `C:\...` などローカルでも記録**（トリガーは保存操作で、格納場所ではない）。Operation は `SensitivityLabelApplied` / `SensitivityLabelUpdated` / `SensitivityLabelRemoved` |
| Office for the web / SharePoint 詳細ペイン / 自動ラベル | ✅ | Operation は `FileSensitivityLabelApplied` / `FileSensitivityLabelChanged` / `FileSensitivityLabelRemoved` |
| Outlook / Exchange | ✅ | メールへのラベルも記録 |
| Microsoft Purview Information Protection **クライアント / スキャナー**（エクスプローラー右クリック等・非 Office ファイル） | ✅（※要確認） | 監査ログのレコード種別は `AipSensitivityLabelAction`（操作名は `SensitivityLabelApplied/Updated/Removed`）。テナントのコネクタ設定により `CloudAppEvents` への流入に差が出る場合があるため、下記で実確認 |

> 本エージェントの `labelOps` には `File*` 系と非 `File*` 系の**両方の Operation を含めてある**ため、
> クラウド側・クライアント側（ローカル操作）双方のラベルイベントを拾います。
>
> ⚠️ 前述の「動作確認」章にある表で `Microsoft Defender for Cloud Apps | No` と記載されるのは、
> **MDCA を Activity Explorer の“情報源”として使えるか**の話であり、`CloudAppEvents` への記録可否とは
> 別物です（混同しないでください）。

**どの経路が実際に流入しているかの実確認**: テスト 2 の結果の `Workload` / `Application` 列を見て、
Word / Excel / SharePoint / OneDrive / Exchange など**どのソースからのラベルイベントが来ているか**を
確認してください。特に**非 Office のローカルファイルを Purview Information Protection クライアントで
ラベル付けした場合**（`AipSensitivityLabelAction`）の流入は環境差が出やすいため、実際にローカルファイルへ
ラベルを付与してから 24–48 時間後にテスト 2 を実行し、該当レコードが現れるかを確認することを推奨します。

---

## エージェント構成

- **種別**: スケジュール実行エージェント（`Interfaces: [Agent]`、`WeeklySchedule` トリガー = 604800 秒）
- **モデル**: `gpt-4o`
- **子スキル**: 7 つの KQL スキル（すべて Target: Defender / CloudAppEvents）＋ 1 つの LogicApp スキル
- **RequiredSkillsets**: `PurviewLabelUsageWeeklyReport`（本マニフェスト自身のインラインスキル）
- **出力**: 自己完結型 HTML レポート（インライン CSS、外部リソース・JavaScript 不使用）を
  `SendLabelReportEmail`（Logic App）でメール配信

---

## レポート項目（出力構成）

1. **対象期間** — 今週／前週の期間・生成日時・付与/変更/削除の総数
2. **エグゼクティブサマリー** — 前週比較・KPI・総合傾向判定バナー（活性 / 安定 / 低調 / 要注意）
3. **ラベル利用状況 集計**（各集計の傾向コメントは最大 1 文・120 文字）
  - 3-1. **ラベル分布**（上位 8、親/子表記・カバレッジ）
   - 3-2. **アクション別集計**（付与 / 変更 / 削除）
  - 3-3. **アプリ／ワークロード別**（上位 6）
  - 3-4. **ユーザー分析**（上位 8）
  - 3-5. **格下げ・削除の注視**（上位 8、旧→新ラベル・理由）
  - 3-6. **ファイル分析**（種別・ファイル名・ディレクトリ、各上位 5）
  - 3-7. **SharePoint サイト別ラベル付与**（上位 7、機密資料の所在把握）

### HTML 出力上限

- HTML 全文: **16,000 文字以内**
- `<style>`: **2,500 文字以内**
- 表のデータ行: レポート全体で **45 行以内**
- 横棒グラフ: 3-1 と 3-2 に限定
- HTML コメント、SVG、base64、JavaScript、装飾専用要素は生成しない
- 必ず `<!DOCTYPE html>` から `</html>` までを含む完全な HTML として送信する

---

## Logic App（メール送信）

同梱の [PurviewLabelUsageWeeklyReport_LogicApp_ARM.json](PurviewLabelUsageWeeklyReport_LogicApp_ARM.json)
をデプロイすると、HTTP トリガー（`manual`）で受け取った `ReportHtml` を Office 365 コネクタで
メール送信する Logic App が作成されます。

- トリガー: `manual`（HTTP Request）— ボディに `ReportHtml` を受け取る
- アクション: **Parse JSON**（`ReportHtml` を抽出）→ **Send an email (V2)**（`Body` に HTML を設定）
- デプロイ後、Office 365 API 接続の認可（配信元メールボックス）を完了させてください。

---

## デプロイ手順

1. [PurviewLabelUsageWeeklyReport_LogicApp_ARM.json](PurviewLabelUsageWeeklyReport_LogicApp_ARM.json) を
   対象リソースグループへデプロイし、Office 365 接続を認可、送信先・件名を設定する。
2. 上記の **動作確認 KQL（テスト 0〜2）** を実行し、ラベル監査イベントが見えることを確認する。
3. **カスタムラベルがある場合**は PowerShell スクリプトで GUID / 親子 / Priority を抽出し、
  [PurviewLabelUsageWeeklyReport.yaml](PurviewLabelUsageWeeklyReport.yaml) の 6 スキルの `LabelMap` に反映する。
4. Security Copilot の **プラグイン管理**で
   [PurviewLabelUsageWeeklyReport.yaml](PurviewLabelUsageWeeklyReport.yaml) をカスタムプラグインとしてアップロードする。
5. プラグイン設定に Logic App の **サブスクリプション ID / リソースグループ / ワークフロー名**を入力する。
6. **アクティブエージェント**でエージェントをセットアップ（認証）し、週次スケジュールで自動実行、
   または手動でレポートを生成・送信する。

---

## 前提条件

| # | 項目 | 内容 |
| --- | --- | --- |
| 1 | ライセンス | Microsoft Purview Information Protection（秘密度ラベル）+ Microsoft Defender XDR（Advanced Hunting） |
| 2 | 統合監査ログ | Purview / Defender の**監査（Unified Audit Log）を有効化**（既定でオンだが要確認） |
| 3 | Defender for Cloud Apps 接続 | Defender ポータル > Settings > Cloud apps > App connectors で **「Microsoft 365 activities」を有効化**。接続直後は取り込みラグあり |
| 4 | 秘密度ラベルの発行 | ラベルポリシーでラベルを発行し、ユーザー／自動ラベルで**実際に付与**されていること |
| 5 | DLP ポリシー | **不要**（本エージェントはラベル監査のみを使用） |

---

## 制約と注意点

- 秘密度ラベルは GUID で記録され、**カスタムラベルは `LabelMap` に登録するまで GUID 表示**です。
- **親 / 子表記**は `LabelMap` の `ParentName` に依存します（`Get-Label` の `ParentId` から投入）。
- **格上げ / 格下げの判定**は旧・新ラベルの `Priority` 比較によるヒューリスティックで、旧ラベル ID が
  取得できないイベントは「変更（不明）」に分類されます。
- Advanced Hunting の**保持期間は約 30 日**（週次用途には十分だが長期保管は別途必要）。
- 本エージェントは**イベント（操作履歴）**を集計します。**現存する全ファイルのラベル分布の棚卸し**には
  Purview の **Content Explorer / Information Protection Reports** を利用してください。
- ラベルが 1 件も見つからない場合、レポート冒頭に注意バナーと「データなし」を明記します
  （架空データは生成しません）。
- HTML はメール送信時の切断を避けるため 16,000 文字以内に制限し、上限へ近づいた場合は
  下位行、グラフ、補足文の順に削減します。

---

## 参考ドキュメント

- [Labeling activities available in Activity explorer](https://learn.microsoft.com/purview/data-classification-activity-explorer-available-events)
- [Audit log activities — Sensitivity label activities](https://learn.microsoft.com/purview/audit-log-activities#sensitivity-label-activities)
- [CloudAppEvents — advanced hunting schema](https://learn.microsoft.com/defender-xdr/advanced-hunting-cloudappevents-table)
- [Connect Microsoft 365 to Microsoft Defender for Cloud Apps](https://learn.microsoft.com/defender-cloud-apps/protect-office-365)
- [Get-Label (Security & Compliance PowerShell)](https://learn.microsoft.com/powershell/module/exchange/get-label)
- [Microsoft Purview posture reports overview](https://learn.microsoft.com/purview/purview-reports)
