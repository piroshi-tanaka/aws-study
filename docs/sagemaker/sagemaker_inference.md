# SageMaker 推論・エッジ推論の全体像まとめ

## 1. どこでモデルを動かすか（全体マップ）

学習は SageMaker（または Python 環境）で行うとして、推論の実行場所は大きく 3 パターンに分けられる。

1. **クラウド（SageMaker マネージド推論）**
   - SageMaker リアルタイム / サーバレスエンドポイント
   - SageMaker Batch Transform（バッチ推論）

2. **クラウド（自己管理インフラ）**
   - EC2 / ECS / EKS / Lambda 上で自前サービスとして推論

3. **エッジ（車載 ECU など）**
   - ONNX Runtime 等を使ってデバイス内で推論
   - 必要に応じて SageMaker Neo ＋ Greengrass / Edge Manager で最適化 & 配布

---

## 2. SageMaker 側の推論オプション

### 2-1. リアルタイム推論エンドポイント（常時起動）

- SageMaker にモデル＋推論コンテナをデプロイすると、**常時起動の HTTPS エンドポイント**が提供される。
- 特徴
  - 低レイテンシのオンライン推論向け（Web / モバイル / API バックエンド）。
  - オートスケーリング、マルチ AZ、A/B テスト（複数モデルへのトラフィック分割）が可能。
  - ユーザーはインスタンスタイプや台数を指定してスケーリングポリシーを設定する。

### 2-2. サーバレス推論（Serverless Inference）

- これも **SageMaker Endpoint の一種**。
- 特徴
  - インスタンスを常時起動せず、リクエストに応じてコンテナを起動する Lambda 的な動作。
  - トラフィックがまばらな場合にコスト効率が良い。
  - コールドスタートによるレイテンシ増加の可能性あり。
  - インスタンスタイプ・台数・スケーリングポリシーを明示的に管理する必要がない。

→ オンライン推論（同期レスポンス）が必要な場合は、  
**常時起動エンドポイント or サーバレスエンドポイントのどちらかを選択**するイメージ。

### 2-3. バッチ推論（Batch Transform）

- S3 上の大量データに対して**一括で推論**する仕組み。
- 特徴
  - エンドポイントは立てない。
  - ジョブ実行中だけインスタンスが立ち上がり、完了後は自動終了。
  - 結果は S3 に書き出される。
  - 夜間バッチや定期再計算などのオフライン処理向き。

→ 「オンライン API で即時レスポンスが欲しい」なら Endpoint、  
「大量データを後処理したい」なら Batch Transform を選ぶ。

---

## 3. Endpoint と API Gateway / Route 53 の関係

- SageMaker Endpoint 自体が HTTPS エンドポイントを持つため、  
  **AWS 内から直接呼び出す場合には API Gateway や Route 53 は必須ではない**。
- ただし実務的には、以下の理由から組み合わせるパターンが多い。
  - API Gateway / ALB で認証・認可（Cognito, JWT 等）やレート制御を一括管理したい。
  - 複数バックエンド（SageMaker, 他のマイクロサービス）を 1 つの REST API として公開したい。
  - Route 53 でカスタムドメインを割り当てたい。

典型構成イメージ：

- クライアント  
  → API Gateway or ALB  
  → Lambda / ECS などのアプリ層（認証・前処理・ロジック）  
  → SageMaker Endpoint（推論）

---

## 4. 自己管理インスタンスでの推論

### 4-1. 使うシーン

- 既に EC2 / ECS / EKS / Lambda ベースのマイクロサービス基盤があり、  
  **監視・ログ・セキュリティ・デプロイなどを統一基準で管理したい**場合。
- 専用ドライバやライセンスなどの制約で、SageMaker Endpoint 上で動かしにくい場合。

### 4-2. 典型フロー

1. SageMaker でモデル学習 → モデルアーティファクトを S3 に保存。
2. 自己管理インフラ（EC2/ECS/EKS/Lambda）のコンテナからモデルをロード。
3. 自前の HTTP API としてマイクロサービス化。

この場合、SageMaker は「学習・実験プラットフォーム」、  
推論は既存システム基盤で運用する、という役割分担になる。

---

## 5. エッジ推論：Neo / Greengrass / ONNX Runtime の役割

### 5-1. ONNX Runtime

- **ONNX 形式のモデルを計算するための推論エンジン（ライブラリ）**。
- C++ / Python / C# / Java / JavaScript などから呼び出すことができる。
- クラウド〜エッジまで、幅広い環境で利用可能。

あなたの想定フロー：

1. scikit-learn でモデル学習。
2. Python で ONNX 形式に変換。
3. ECU 上の C++ ＋ ONNX Runtime で推論。

→ この構成だけで、Greengrass / Neo 不使用でもエッジ推論は実現できる。

### 5-2. SageMaker Neo

- 役割：**学習済みモデルをターゲットデバイス向けに最適化・コンパイルするコンパイラ的なサービス**。
- 対応フレームワーク：
  - TensorFlow / PyTorch / XGBoost / scikit-learn / ONNX など。
- 対象ハードウェア：
  - ARM / x86 / NVIDIA GPU / NXP などの CPU / SoC。
- 効果：
  - 推論速度向上、メモリ削減、バイナリサイズ縮小など。

利用タイミング：

1. 学習済みモデルを Neo に投入。
2. ターゲット（例：ARM Linux, NVIDIA Jetson 等）を指定してコンパイル。
3. 出力された最適化済みモデルをクラウドまたはエッジにデプロイ。

### 5-3. AWS IoT Greengrass

- 役割：**エッジデバイスにアプリケーションや ML モデルを配布・起動・更新するためのランタイム＋クラウドサービス**。
- エッジ側：
  - Greengrass Core ソフトウェア（ランタイム）を Linux / Windows デバイスにインストール。
  - その中で「コンポーネント」と呼ばれる単位でアプリ（コンテナ / Lambda ライクなコード / ネイティブバイナリ）を実行。
- クラウド側：
  - どのデバイスにどのコンポーネントをどのバージョンで配布するかを管理。
  - デバイスからのログ・メトリクス収集。

構造イメージ（レイヤ）：

- ECU の OS / ハードウェア
- その上に Greengrass Core ランタイム（ユーザーがインストール）
- その上で Greengrass コンポーネントとして、
  - あなたが作る C++ / Python アプリ
  - モデルファイル（ONNX or Neo 最適化版）
  を実行
- アプリ内では ONNX Runtime や Neo のランタイムを呼び出して推論する

→ **Greengrass は「ONNX Runtime の代わり」ではなく、  
「ONNX Runtime を使うアプリをエッジに配って動かす基盤」**という位置づけ。

---

## 6. エッジ構成のパターン整理（車載 ECU 想定）

### パターン A：ECU が Linux / コンテナ対応で、AWS IoT 連携を前提

- Neo を使う：
  - scikit-learn → Neo でターゲット向けに最適化。
  - モデル＋推論コードを Greengrass コンポーネントとしてパッケージ。
  - AWS IoT Core / Greengrass から各 ECU に配布・更新。
- Greengrass Core が各 ECU 上でアプリを起動し、ローカルで推論実行。

### パターン B：ECU が RTOS / 独自環境で、AWS ランタイムを載せられない

- Greengrass / Neo を使わない構成：
  - scikit-learn → ONNX 変換（Python）。
  - ECU 側に C++ ＋ ONNX Runtime を組み込み。
  - モデルファイル（ONNX）の配布・更新は OEM 独自の OTA やフラッシュ書き換え手段で実施。
- この場合、AWS は主に「クラウド側の学習・実験基盤」として利用し、  
  デバイス側は OEM の制約・仕組みに合わせて実装する。

---

## 7. まとめ

- **SageMaker Endpoint**
  - オンライン推論用の基本の選択肢（常時起動 / サーバレス）。
  - API Gateway 等は必須ではないが、認証・統合 API の観点で組み合わせることが多い。
- **Batch Transform**
  - S3 上の大量データに対するオフライン一括推論。
- **自己管理インスタンス**
  - 既存の EC2/ECS/EKS/Lambda 基盤に乗せて、他マイクロサービスと同じルールで運用したい場合の選択肢。
- **エッジ推論**
  - ONNX Runtime：モデルを実際に推論するライブラリ。
  - SageMaker Neo：モデルをターゲット HW 向けに最適化するコンパイラ。
  - AWS IoT Greengrass：エッジデバイス上でアプリ・モデルを配布・起動・更新するためのランタイム＋管理サービス。
- 車載 ECU で Neo / Greengrass を使うかどうかは、
  - ECU の OS（Linux か RTOS か）
  - コンテナ対応の有無
  - AWS IoT と連携したいかどうか
  といった前提条件で決まる。

# 参考リンク一覧

## SageMaker 推論関連

- SageMaker リアルタイム推論（英語）
  - https://docs.aws.amazon.com/sagemaker/latest/dg/realtime-endpoints.html
- SageMaker リアルタイム推論（日本語）
  - https://docs.aws.amazon.com/ja_jp/sagemaker/latest/dg/realtime-endpoints.html

- SageMaker Serverless Inference（英語）
  - https://docs.aws.amazon.com/sagemaker/latest/dg/serverless-endpoints.html
- SageMaker Serverless Inference（日本語）
  - https://docs.aws.amazon.com/ja_jp/sagemaker/latest/dg/serverless-endpoints.html

- SageMaker Batch Transform（英語）
  - https://docs.aws.amazon.com/sagemaker/latest/dg/batch-transform.html
- SageMaker Batch Transform 解説ブログ（日本語）
  - https://tech.fusic.co.jp/posts/2022-09-06-aws-batch-transform/
  - https://zenn.dev/megazone_jp/articles/b745710d61c447

- SageMaker エンドポイント管理（日本語）
  - https://docs.aws.amazon.com/ja_jp/sagemaker/latest/dg/realtime-endpoints-manage.html
- リアルタイムエンドポイントの呼び出し方法（日本語）
  - https://docs.aws.amazon.com/ja_jp/sagemaker/latest/dg/realtime-endpoints-test-endpoints.html

## SageMaker Neo / Edge 関連

- SageMaker Neo 概要（英語）
  - https://docs.aws.amazon.com/sagemaker/latest/dg/neo.html
- SageMaker Neo 概要（日本語）
  - https://docs.aws.amazon.com/ja_jp/sagemaker/latest/dg/neo.html
- Neo によるモデルコンパイル（日本語）
  - https://docs.aws.amazon.com/ja_jp/sagemaker/latest/dg/neo-job-compilation.html
- Neo 対応エッジデバイス一覧（日本語）
  - https://docs.aws.amazon.com/ja_jp/sagemaker/latest/dg/neo-edge-devices.html

## AWS IoT Greengrass 関連

- AWS IoT Greengrass とは（英語）
  - https://docs.aws.amazon.com/greengrass/v2/developerguide/what-is-iot-greengrass.html
- AWS IoT Greengrass の仕組み（英語）
  - https://docs.aws.amazon.com/greengrass/v2/developerguide/how-it-works.html
- AWS IoT Greengrass V2 入門チュートリアル（日本語）
  - https://docs.aws.amazon.com/ja_jp/greengrass/v2/developerguide/getting-started.html

- Greengrass V2 解説（日本語ブログ・DevelopersIO）
  - Greengrass V2 の使い方の基本  
    https://dev.classmethod.jp/articles/aws-iot-greengrass-v2-basics/
  - AWS入門ブログリレー2024〜AWS IoT Greengrass V2 編  
    https://dev.classmethod.jp/articles/introduction-2024-aws-iot-greengrass-v2/

## ONNX / ONNX Runtime 関連

- ONNX Runtime 公式サイト
  - https://onnxruntime.ai/
- ONNX Runtime ドキュメント
  - https://onnxruntime.ai/docs/
- ONNX Runtime インストールガイド
  - https://onnxruntime.ai/docs/install/

- ONNX 公式サイト
  - https://onnx.ai/
