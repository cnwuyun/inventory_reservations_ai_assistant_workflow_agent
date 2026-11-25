# Inventory Reservations AI Assistant (Workflow Agent)

> Oracle Fusion SCM 在庫管理領域の AI Agent — プリビルト Agent を Workflow Agent に再構築し、安定性を大幅に改善。

---

## 概要

この AI Agent は、Oracle Fusion Supply Chain Management (SCM) の **在庫管理 (Inventory Management)** 領域において、在庫予約（Reservation）の作成・確認・更新・削除を自然言語で行うためのアシスタントです。

元となる Oracle プリビルト Agent `ORA_RESERVATIONSWORKER_V10` を、**Workflow Agent アーキテクチャ** で再構築し、従来の「単一 LLM Agent」方式と比較して以下の点が改善されています。

## Prebuilt Agent からの主な改善点

| 項目 | Prebuilt Agent（単一 LLM） | 本 Workflow Agent |
|---|---|---|
| アーキテクチャ | 1 つの LLM が全ての処理を実行 | LLM は意図抽出・応答生成のみ、各種 API 呼び出しは Workflow の決定論的ノードが担当 |
| 安定性 | LLM の出力に依存するため、パラメータ不足・順序誤りが発生しうる | Tool / Business Object ノードで全パラメータを明示的にバインド。型エラーやリクエスト欠落を防止 |
| 再現性 | 同一リクエストでも LLM の出力が変動 | 条件分岐とループで処理フローが固定化され、結果が一貫 |
| デバッグ | LLM 内部の挙動を追跡困難 | Workflow ノードごとに入出力を検証可能 |
| 機能 | 予約作成・確認・更新・削除 | 上記に加え、**2 オーダー同時照会**、**削除＋再引当（DeleteAndReallocate）** などの複合フローを追加実装 |

## ファイル構成

```
yamaguchi/
├── README.md
├── env.properties                           # AI Studio CLI 接続設定（適宜変更）
├── package.json
└── src/
    ├── agents/
    │   └── reservationsworker_v10_yy.agent                  # 元 Prebuilt Agent（参考）
    ├── workflows/
    │   ├── inventory_reservations_ai_assistant_single_agent_v10_yy.wf   # 単一 Agent 版（参考）
    │   └── inventory_reservations_ai_assistant_workflow_agent_v1_yy.wf  # ★ 本編：Workflow Agent
    ├── businessObjects/                     # 予約・在庫関連 BO 定義（11 本）
    ├── tools/                               # 予約操作用 Tool 定義（13 本）
    └── deeplinks/                           # 画面遷移用 DeepLink 定義（2 本）
```

### 各ファイルの用途

| ディレクトリ | 概要 |
|---|---|
| `agents/` | 元の Prebuilt Agent 定義。Workflow Agent では直接使用しませんが、ツール・BO の参照元です。 |
| `workflows/` | 本体の Workflow 定義。`workflow_agent` 版が本プロジェクトの主要成果物です。 |
| `businessObjects/` | Oracle SCM Inventory Management の REST API をラップした BO 定義。予約明細、需要/供給ドキュメントなど。 |
| `tools/` | Agent が呼び出すツール定義。予約作成・削除・照会、ユーザーセッション取得など。 |
| `deeplinks/` | 予約一覧画面・需要ドキュメント画面への DeepLink 生成ツール。 |

## 前提条件

- **Oracle AI Studio**（AI Agent Studio）2025 年以降のバージョン
- 対象 Oracle Fusion Applications 環境: **SCM / Inventory Management**（24D 以降を推奨）
- AI Studio CLI がローカルにインストール済み

## インストール手順

1. **ZIP パッケージをダウンロード**
   GitHub Releases より `inventory-reservations-workflow-agent.zip` を取得してください。

2. **AI Studio に Import**
   ```
   ai-studio import --file inventory-reservations-workflow-agent.zip --type workflow
   ```
   または、AI Studio コンソール → **Workflows** → **Import** よりアップロードしてください。

3. **認証情報を設定**
   AI Studio コンソール上で、各 Business Object / Tool に対応する接続先（Fusion 環境の FAS Host、Oauth クライアント等）を設定してください。

4. **Workflow を有効化**
   Import 完了後、ステータスを **Active** に切り替えてください。

## 使用例

自然言語による指示例：

| ユーザー発話 | 実行される処理 |
|---|---|
| `オーダー 98584 の予約状況を教えて` | 予約照会（ViewReservations） |
| `98580 の予約を削除して 98582 に引き当てて` | 削除＋再引当（DeleteAndReallocate） |
| `オーダー 98584 と 98585 の予約を同時に一覧で見たい` | 2 オーダー同時照会 |
| `品番 ABC001 の在庫を M1 倉庫で確認して` | 在庫アイテム照会 |
| `Sales Order 12345 の明細を予約して` | 需要伝票明細から予約作成 |

## 開発者向け

### ローカルテスト実行

```bash
# Workflow テスト（ローカル判定）
npm run test:workflows:file

# ライブ環境テスト
npm run test:workflows:live
```

### 再 Export 手順

```bash
# AI Studio CLI で Workflow を Export
ai-studio export --workflow INVENTORY_RESERVATIONS_AI_ASSISTANT_WORKFLOW_AGENT_V1_YY \
                 --output dist/inventory-reservations-workflow-agent.zip
```
