# コード構造の分析レポート

## 1. 全体アーキテクチャと設計思想

n8nのソフトウェアアーキテクチャは、**関心の分離 (Separation of Concerns)** という基本原則に強く基づいている。コードベースは、それぞれが明確な責務を持つ独立したパッケージ群（モノレポ構成）に分割されている。これにより、各機能は疎結合に保たれ、変更や拡張が他の部分に予期せぬ影響を与えにくくなっている。

主要なアーキテクチャパターンは以下の通りである。

-   **バックエンド**:
    *   **静的定義 vs 動的実行**: ワークフローの「設計図」としての静的な構造（`n8n-workflow`）と、その設計図を実際に動かす「実行エンジン」（`n8n-core`）が明確に分離されている。
    *   **拡張可能なノードシステム**: 外部サービスとの連携機能（ノード）は、`INodeType`という共通のインターフェース規約に従うことで、プラグインのように追加・拡張が可能になっている（`nodes-base`）。
-   **フロントエンド**:
    *   **コンポーネントベースアーキテクチャ**: UIは再利用可能なコンポーネント（Vue.js）の組み合わせで構築されている。
    *   **状態管理の一元化**: アプリケーション全体の状態はPiniaストアに集約され、単一の信頼できる情報源（Single Source of Truth）として機能する。
    *   **API層の抽象化**: バックエンドとの通信は専用のAPIクライアントにカプセル化され、UIコンポーネントはHTTP通信の詳細を意識する必要がない。

この設計により、開発者は特定の課題領域（例: 新規ノードの開発、UIの改善、実行エンジンのパフォーマンスチューニング）に集中でき、コードベース全体の保守性とスケーラビリティが確保されている。

## 2. バックエンド分析

### 2.1. `n8n-workflow`: ワークフローの静的定義

このパッケージは、ワークフローが「どのようなものであるか」という静的な構造（設計図）を定義し、操作するための責務を持つ。

#### `Workflow` クラス (`workflow.ts`)

ワークフローの定義全体をカプセル化する中心的なクラス。

-   **責務**: ノード、接続、設定といったワークフローの構成要素をオブジェクトモデルとして保持し、それらを操作するためのAPIを提供する。
-   **データ構造の工夫**:
    *   `nodes`: 単なる配列ではなく、ノード名をキーとするオブジェクト (`{ [nodeName]: INode }`) としてノードを保持する。これにより、`O(1)`の計算量で特定のノードに高速アクセスできる。
    *   `connectionsBySourceNode` / `connectionsByDestinationNode`: ノード間の接続情報を、出発点（ソース）と到達点（デスティネーション）の両方をキーとして保持する。これにより、あるノードから下流（子孫）をたどる処理と、上流（祖先）をたどる処理の両方を効率的に行える。
-   **重要なメソッド**:
    *   `renameNode(currentName, newName)`: 単にノード名を変更するだけでなく、他のノードのパラメータで使われている式 (`={{...}}`) の中の参照も自動で更新する。これにより、UI上でのノード名変更が、ワークフロー全体の整合性を壊すことなく安全に行われる。
    *   `getStartNode()`: ワークフローに複数のトリガーノードが存在する場合や、トリガーが存在しない場合でも、定義されたルール（例: `STARTING_NODE_TYPES`の優先順位）に基づき、実行を開始すべき最適なノードを自動的に判定する。

#### `Expression` クラス (`expression.ts`)

ノードのパラメータ値として埋め込まれる動的な式（例: `={{ $json.id }}`）を評価する責務を持つ。

-   **責務**: 式の安全な評価環境（サンドボックス）を提供し、実行時のデータコンテキストに基づいて式を評価する。
-   **サンドボックス化**: `initializeGlobalContext`メソッドは、式の実行環境で利用可能なグローバルオブジェクトや関数を厳格に制限する。`eval`, `fetch`, `setTimeout`といった潜在的に危険な関数は無効化され、`Date`, `Math`, `JSON`といった安全なユーティリティと、n8n独自のヘルパー関数のみが許可される（許可リスト方式）。これにより、悪意のある式がサーバー上で任意のコードを実行することを防いでいる。
-   **`WorkflowDataProxy`との連携**: `resolveSimpleParameterValue`メソッド内で`WorkflowDataProxy`をインスタンス化する。このプロキシオブジェクトが、式の中から`$json`や`$node`といった特別な変数を通じて、現在の実行コンテキストのデータ（上流ノードの実行結果など）にアクセスするための架け橋となる。

### 2.2. `n8n-core`: ワークフローの動的実行

このパッケージは、`n8n-workflow`によって定義された静的な「設計図」を受け取り、それを実際に動かすランタイムエンジンとしての責務を持つ。

#### `WorkflowExecute` クラス (`execution-engine/workflow-execute.ts`)

ワークフロー実行のライフサイクル全体を管理する、まさに心臓部と言えるクラス。

-   **責務**: ワークフローの開始から終了までの一連のプロセスを制御するステートマシン。
-   **実行ループ (`processRunExecutionData`)**:
    1.  実行すべきノードを管理するキュー（`nodeExecutionStack`）からノードを一つ取り出す。
    2.  `runNode`メソッドを呼び出し、そのノードの`execute`ロジックを実行する。
    3.  実行結果（成功データまたはエラー）を受け取る。
    4.  `addNodeToBeExecuted`メソッドを呼び出し、実行結果を後続ノードの入力として接続し、後続ノードをキューに追加する。
    5.  キューが空になるまで1〜4を繰り返す。
-   **エラーハンドリング**: `runNode`内の`try...catch`ブロックが中心的な役割を担う。ノード実行時にエラーが発生した場合、それを捕捉し、実行データにエラー情報を記録する。`continueOnFail`が有効であれば、後続ノードの実行を継続し、無効であればワークフロー全体の実行を停止する。
-   **非同期処理とキャンセル**: 実行プロセス全体は`PCancelable`（キャンセル可能なPromise）でラップされている。これにより、ユーザーがUIから実行を中断した場合や、設定されたタイムアウトに達した場合に、進行中のHTTPリクエストなどを安全に中断し、実行を停止させることができる。

### 2.3. `nodes-base`: 拡張性の基盤

外部サービスとの連携機能（ノード）の実装が集約されたパッケージ。n8nの拡張性の核となる。

#### `INodeType` インターフェース

すべてのノードが実装すべき共通規約。

-   **`description` プロパティ（静的定義）**:
    *   **責務**: ノードのUIとメタデータを定義する。この情報はフロントエンドの`editor-ui`で解釈され、ノードのプロパティパネルを描画するために使われる。
    *   **`properties`**: ノードがユーザーに要求するパラメータのリスト。各要素がUI上の入力フィールド（テキストボックス、ドロップダウン、トグルスイッチなど）に対応する。`displayName`, `name`, `type`, `default`といったキーを持つ。`displayOptions`を使えば、他のパラメータの値に応じて特定のプロパティを表示・非表示にする動的なUIも実現できる。
-   **`execute` メソッド（動的ロジック）**:
    *   **責務**: ワークフロー実行時に呼び出され、ノードの実際の処理を行う。
    *   **実行コンテキスト (`this` as `IExecuteFunctions`)**: 実行エンジンから提供されるヘルパー関数群へのアクセスポイント。これにより、ノード開発者は定型的な処理を簡潔に記述できる。
        *   `getInputData()`: 上流ノードからの入力データを取得。
        *   `getNodeParameter()`: ユーザーがUIで設定したパラメータ値を取得。式が使われている場合、このメソッドを呼んだ時点ですでに評価済みの値が返る。
        *   `getCredentials()`: 実行に必要な認証情報を安全に取得。
        *   `this.helpers.requestWithAuthentication()`: 認証情報の自動付与、OAuth2のトークンリフレッシュ、ページネーション、リトライといった複雑な処理を内部で吸収してくれる高レベルなHTTPリクエスト関数。これにより、ノード開発者はAPIの仕様に集中できる。

### 2.4. ワークフロー実行のライフサイクルとシーケンス

ここまでの分析を基に、ユーザーがUIで「実行ボタン」をクリックしてから、ワークフローが実行され、結果がUIに返却されるまでの一連の流れをシーケンス図として示す。

```mermaid
sequenceDiagram
    actor User
    participant FE as Browser (Vue App)
    participant API as Backend (API Layer)
    participant Runner as Backend (WorkflowRunner)
    participant Engine as Backend (WorkflowExecute)

    User->>+FE: 1. 実行ボタンをクリック (in NodeView.vue)
    FE->>FE: 2. useRunWorkflow.runEntireWorkflow()
    FE->>FE: 3. workflowsStore.runWorkflow()
    FE->>+API: 4. POST /workflows/{id}/run
    note right of FE: (HTTP Request)

    API->>API: 5. WorkflowsController.runManually()
    API->>+Runner: 6. WorkflowExecutionService.executeManually()
    Runner->>+Engine: 7. new WorkflowExecute()
    Engine->>Engine: 8. processRunExecutionData()

    loop 実行キューが空になるまで
        Engine->>Engine: 9. runNode()
        note over Engine: ノードのexecute()を実行し、<br>HTTPリクエストなどを外部へ送信
        Engine-->>-Runner: 10. [Hook] nodeExecuteAfter
        Runner-->> API: 11. WebSocketService.send()
        API-->>-FE: 12. [WebSocket] 実行状況を送信
        FE->>FE: 13. ストアを更新しUIに反映
    end
    note right of API: (リアルタイム更新)

    Engine-->>-Runner: 14. 最終実行結果 (Promise解決)
    Runner-->>-API: 15.
    API-->>-FE: 16. HTTP 200 OK
    deactivate FE
```

## 3. フロントエンド分析 (`frontend/editor-ui`)

Vue.js 3をベースに構築された、ワークフローを視覚的に作成・編集するためのシングルページアプリケーション（SPA）。

### 3.1. アーキテクチャとディレクトリ構造

`src/app/`配下は、責務ごとに明確にディレクトリが分割されている。

-   **`views/`**: 特定のURLに対応するページレベルのコンポーネント。アプリケーションの主要な画面（ワークフローエディタ、認証情報一覧など）を構成する。`NodeView.vue`がエディタ画面のルートとなる。
-   **`components/`**: `views`を構成する、再利用可能なより小さなUI部品。`WorkflowCanvas.vue`, `NodeDetailsView.vue`などが含まれる。
-   **`stores/` (`Pinia`)**: アプリケーション全体のグローバルな状態管理。
    *   `workflowsStore`: 現在編集中のワークフローのノード、接続、設定などの状態を一元管理。ノードの追加・削除、パラメータの変更といった操作は、このストアのアクションを通じて行われる。
    *   `ndvStore (Node Details View)`: 現在選択されているノードや、ノード詳細パネルの表示状態を管理する。
-   **`api/` (-> `packages/client/src/`)**: バックエンドとのREST API通信をすべてカプセル化する層。`axios`をベースに、エンドポイントごとのクライアントが定義されている。UIコンポーネントやストアは、この層を通じてバックエンドと通信するため、HTTPリクエストの詳細から分離される。
-   **`composables/`**: Vue 3のComposition APIを用いた再利用可能なロジック。`useRunWorkflow.ts`のように、特定の機能に関するリアクティブな状態とメソッドをカプセル化する。

### 3.2. ワークフローキャンバスの描画

-   **`@vue-flow/core`**: ワークフローエディタのキャンバス描画の基盤となるライブラリ。ノードとエッジ（接続）のデータをリアクティブに受け取り、SVGベースでレンダリングする。
-   **データフロー**:
    1.  `workflowsStore`が保持するノードと接続のデータを、`@vue-flow/core`が要求するフォーマットに変換する。
    2.  変換されたデータが`<VueFlow>`コンポーネントにプロパティとして渡される。
    3.  ユーザーがキャンバス上でノードをドラッグ＆ドロップしたり、接続を作成したりすると、`@vue-flow/core`がイベント（例: `onNodeDragStop`, `onConnect`）を発行する。
    4.  これらのイベントを捕捉し、`workflowsStore`のアクションを呼び出すことで、アプリケーションの状態を更新する。
    5.  Piniaのリアクティビティにより、状態の変更は自動的に`<VueFlow>`コンポーネントに反映され、UIが再描画される。この単方向データフローが、UIの状態管理をシンプルに保っている。

### 3.3. 主要コンポーネントと状態管理の関係

ワークフローエディタ画面 (`NodeView.vue`) における主要コンポーネントとPiniaストア間のインタラクションを以下に図示する。

```mermaid
componentDiagram
    title Workflow Editor Component Architecture

    package "Pinia Stores" {
        [workflowsStore]
        [ndvStore]
    }

    package "Views (NodeView.vue)" {
        [WorkflowCanvas]
        [FocusPanel]
        [NodeCreation]
        [NodeDetailsView]
    }

    actor User

    User -->> WorkflowCanvas : (1) Click Node
    User -->> NodeDetailsView : (4) Edit Parameters
    User -->> WorkflowCanvas : Add Node

    WorkflowCanvas -->> ndvStore : (2) Set Active Node
    ndvStore -->> FocusPanel : (3) Show Panel
    ndvStore -->> NodeDetailsView : (3) Show Details
    NodeDetailsView -->> workflowsStore : (5) Update Node Parameters
    workflowsStore -->> WorkflowCanvas : (6) Re-render Node

    WorkflowCanvas -->> NodeCreation : Open Panel
    NodeCreation -->> workflowsStore : Add New Node
    workflowsStore -- -->> WorkflowCanvas : Render New Node
```

**インタラクションの解説:**

1.  **ユーザー**が`WorkflowCanvas`上のノードをクリックする。
2.  `WorkflowCanvas`は`ndvStore`のアクションを呼び出し、クリックされたノードを「アクティブなノード」として設定する。
3.  `ndvStore`の状態変更を検知した`FocusPanel`と`NodeDetailsView`が自身を表示状態にし、アクティブなノードの詳細を描画する。
4.  **ユーザー**が`NodeDetailsView`内のフォームでパラメータを編集する。
5.  `NodeDetailsView`は入力イベントを捕捉し、`workflowsStore`のアクションを呼び出して、対応するノードのデータを更新する。
6.  `workflowsStore`の状態変更はリアクティブに`WorkflowCanvas`に伝播し、表示されているノードの情報（例: ラベル）が自動的に更新される。

このように、ユーザー操作は各コンポーネントで捕捉されるが、状態の変更は必ずPiniaストアを介して行われる。これにより、コンポーネント間の直接的な依存関係が排除され、アプリケーション全体の見通しが良くなっている。
