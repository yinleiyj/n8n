# パッケージ: `n8n-core`

## 1. 概要

`n8n-core`パッケージは、n8nアプリケーションの**ランタイムエンジン**です。`n8n-workflow`パッケージによって定義された静的な「設計図」を受け取り、それを実際に一つずつ実行していく責務を担います。ワークフローの「動的な実行」に関するすべてのロジックがこのパッケージに集約されています。

その他、資格情報（Credentials）の管理や、ノード定義の動的なロードなど、実行に必要なコアサービスも提供します。

## 2. 主要なコンセプトとアーキテクチャ

`n8n-core`のアーキテクチャは、`WorkflowExecute`クラスを中心に構築されています。

### 2.1. `WorkflowExecute` クラス (`execution-engine/workflow-execute.ts`)

ワークフロー実行のライフサイクル全体を管理する、まさに心臓部と言えるクラスです。ステートマシンのように振る舞い、ワークフローの開始から終了までの一連のプロセスを制御します。

-   **責務:**
    -   実行すべきノードを管理するキュー（`nodeExecutionStack`）を保持し、それをループ処理で一つずつ実行します。
    -   各ノードの実行結果（成功データまたはエラー）を`runExecutionData`オブジェクトに記録します。
    -   実行結果を後続ノードの入力として渡し、次のノードをキューに追加します。
    -   複数入力を持つノード（例: Mergeノード）のために、すべての入力が揃うまでデータを一時的に保持（`waitingExecution`）し、実行を待機させます。
    -   ノードの`retryOnFail`や`continueOnFail`といった設定に基づき、エラーハンドリングやリトライ処理を実行します。

-   **実行ループ (`processRunExecutionData` メソッド内):**
    1.  `nodeExecutionStack`から実行すべきノード情報（`IExecuteData`）を一つ取り出す。
    2.  `runNode`メソッドを呼び出し、そのノードの`execute`ロジックを実行する。
    3.  実行結果（成功データまたはエラー）を受け取る。
    4.  `addNodeToBeExecuted`メソッドを呼び出し、実行結果を後続ノードの入力として接続し、後続ノードをキューに追加する。
    5.  キューが空になるか、エラーで停止するまで1〜4を繰り返す。

### 2.2. 資格情報 (Credentials) の管理 (`credentials.ts`)

外部サービスへの接続に必要な認証情報（APIキー、OAuthトークンなど）を安全に管理する責務を持ちます。

-   **暗号化・復号化:** データベースに保存される資格情報は、環境変数 `N8N_ENCRYPTION_KEY` を用いて暗号化されます。`Credentials.getDecrypted()`メソッドを通じて、ノード実行時に初めて復号化され、安全に利用されます。
-   **動的ロード:** `Credentials.getByName()`メソッドは、実行時に必要な資格情報をデータベースから動的にロードします。

## 3. ディレクトリ構造と主要ファイル

`packages/core/src/`ディレクトリ下の主要なファイルと責務は以下の通りです。

```
packages/core/src/
├── execution-engine/      # ワークフロー実行のコアロジック
│   ├── workflow-execute.ts  # WorkflowExecuteクラスの定義。実行ライフサイクルの中心。
│   └── partial-execution-utils/ # UIからの部分実行（Execute Node）に関連するユーティリティ。
├── credentials.ts         # 資格情報の暗号化、復号化、管理を行うクラス。
├── encryption/            # 暗号化アルゴリズムに関する低レベルな実装。
├── nodes-loader/          # `nodes-base`などからノード定義を動的に読み込む責務。
├── interfaces.ts          # パッケージ内で使用される型定義。
└── node-execute-functions.ts # ノードの`execute`メソッドに渡されるヘルパー関数群の実装。
```

### 3.1. 主要ファイルの詳細

-   **`execution-engine/workflow-execute.ts`**:
    -   `WorkflowExecute`クラスを定義します。
    -   `run`メソッドが手動実行の、`runPartialWorkflow2`がUIからの部分実行のエントリーポイントです。
    -   `processRunExecutionData`メソッドが、`nodeExecutionStack`を消費していくメインの実行ループを含みます。

-   **`credentials.ts`**:
    -   `Credentials`クラスを定義します。
    -   データベースから資格情報を取得し、`encryption`ディレクトリのユーティリティを使って復号化する責務を持ちます。ノードからは直接利用されず、実行コンテキストを通じて間接的に利用されます。

-   **`node-execute-functions.ts`**:
    -   ノードの`execute`メソッド内で`this`として参照されるヘルパー関数群（`getCredentials`, `getNodeParameter`, `getInputData`など）の実体を定義しています。
    -   `WorkflowExecute`は、これらの関数に必要なコンテキスト（現在のノード、実行データなど）を注入した`ExecuteContext`オブジェクトを生成し、各ノードに渡します。

## 4. 他パッケージとの関連

```mermaid
graph TD
    subgraph n8n-workflow
        Workflow
    end

    subgraph nodes-base
        NodeTypeImplementation[Node Type Implementation<br>(e.g., GoogleSheets.node.ts)]
    end

    subgraph n8n-core
        WorkflowExecute
        Credentials
        NodeExecuteFunctions
    end

    WorkflowExecute -- "Reads structure from" --> Workflow
    WorkflowExecute -- "Executes logic of" --> NodeTypeImplementation
    NodeTypeImplementation -- "Uses helpers from" --> NodeExecuteFunctions
    NodeExecuteFunctions -- "Accesses secure data via" --> Credentials
```

-   **`n8n-workflow`**: `WorkflowExecute`は、`n8n-workflow`が提供する`Workflow`オブジェクトを読み込み、どのノードをどの順番で実行すべきかを判断します。
-   **`nodes-base`**: `WorkflowExecute`は、`nodes-loader`を通じて読み込んだ各ノードの`execute`メソッドを呼び出します。ノード側は、`n8n-core`が提供する`NodeExecuteFunctions`（実行コンテキスト）を通じて、パラメータの取得や資格情報の復号化といったコア機能を利用します。

## 5. 開発者向けガイド

-   **実行ロジックのデバッグ**: ワークフローが期待通りに動かない場合、まず追うべきは`WorkflowExecute.ts`です。ブレークポイントを`processRunExecutionData`のループ開始地点や、`runNode`メソッドの冒頭に設定することで、各ステップでの`runExecutionData`（特に`nodeExecutionStack`と`resultData.runData`）の内容を監視するのが非常に有効です。
-   **複数入力ノードの挙動**: Mergeノードなどがうまく動かない場合は、`addNodeToBeExecuted`メソッド内の`waitingExecution`オブジェクトがどのように変化するかを確認してください。データが期待通りに待機状態になっているか、あるいは意図せず実行キューに追加されてしまっていないかを調査します。
-   **カスタムヘルパー関数の追加**: すべてのノードから利用できる新しいヘルパー関数を追加したい場合は、`node-execute-functions.ts`と、それにコンテキストを渡す`execution-engine/node-execution-context/execute-context.ts`を拡張することを検討してください。
