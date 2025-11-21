# 1. 本練習プロジェクトの概要
AWSのサーバーレスサービスを活用し、画像がアップロードされたら自動的にリサイズ処理を行い、完了通知をメールで送信するシステムを構築する。

本プロジェクトは、生成AIを技術ペアプログラマーとして活用した実践演習として実施。 IAMアカウント作成からPythonコードの実装、トラブルシューティングまで、AIとの対話を通じて遂行した。

# 2. 利用サービス
  * **Compute**: AWS Lambda (Python 3.12 / x86\_64)
  * **Storage**: Amazon S3 (Input Bucket, Output Bucket)
  * **Security**: IAM (Roles & Policies), S3 Block Public Access
  * **Dev Tools**: AWS CloudShell (Linux環境でのデプロイ), Boto3 (AWS SDK), Pillow (画像処理ライブラリ)

# 3. 簡単な流れ
1.  **S3**: 入力用と出力用のバケットを2つ作成する。
1.  **IAM**: LambdaがS3とSNSを操作できる権限（ロール）を作成する。
1.  **Lambda**: 関数の枠を作成し、タイムアウト等の設定を行う。
1.  **Deploy**: AWS CloudShellを使い、コードとライブラリをパッケージ化してデプロイする。
1.  **Trigger**: S3へのアップロードをトリガーにLambdaが動くよう設定する。


# 4. 詳細手順

## 4.1 AWS　アカウント作成

### 4.1.1 AWS IAMユーザー作成
前提：ルートユーザーの作成が完了したこと
1. ルートユーザーでログインする
1. IAMダッシュボードへ移動し、AWSコンソールの検索窓で「IAM」と検索し、サービスを開きます
1. 左メニューの「ユーザー」をクリック → 「ユーザーの作成」 オレンジ色のボタンをクリック
1. ユーザー名: 「HaoAdmin」と入力
1. 「AWSマネジメントコンソールへのユーザーアクセスを提供する」にチェックを入れる
1. コンソールパスワード: 「カスタムパスワード」を選び、自分でパスワードを設定（または自動生成）
1. 「次へ」をクリック
1. 許可の設定をする
    * 許可のオプション: 「ポリシーを直接アタッチする」 を選択
    * 許可ポリシーのリストから: 検索窓に Admin と入力
    * 「AdministratorAccess」というポリシーが表示されるので、チェックを入れる  
        ※ こちらのポリシーは、「請求情報以外はほぼ何でもできる管理者権限」です。
1. 「次へ」をクリック
1. 内容を確認して 「ユーザーの作成」 をクリック
1. ログイン情報の保存:
    * .csv ファイルをダウンロードするボタンが表示され、ダウンロードして保管する

### 4.1.2 セキュリティー設定（二段階認証MFA）
前提：IAMユーザーでログインし、MFAを設定する
1. 作成したIAMユーザーでログインし直す
2. 右上のユーザー名をクリック → 「セキュリティ認証情報」
3. 「多要素認証 (MFA)」の 「MFA デバイスの割り当て」 から設定
4. デバイスを識別できるように、「デバイス名」を入力(自分のMacを利用したので、MyMacで入力した)
5. パスキーを選択し、MacBookの指紋認証を行う
6. 設定完了

## 4.2 S3バケット作成（入力用と出力用）
1. AWSコンソールで S3 を開く
1. 「バケットを作成」をクリック
1. 「バケット名」に「hao-image-resize-input」
1. 他の設定は、デフォルトのままで問題なし
1. 同様に「hao-image-resize-output」のバケットを作成

## 4.3 IAMロールの作成（権限設定）
概要:  
　　Lambdaが「S3から画像を読み込む」「S3に画像を書き込む」「ログを出す」ための許可証（ロール）を作る

手順：  
1. AWSコンソールで IAM を開く -> ロール -> ロールを作成
1. ステップ1:
    * 信頼されたエンティティタイプ: AWSのサービス
    * ユースケース: Lambda を選択して「次へ」
1. ステップ2:
    * 許可ポリシーで以下を検索してチェックを入れる
        * AWSLambdaBasicExecutionRole (ログ出力用)
        * AmazonS3FullAccess(S3操作用)  
        ※ 本来は特定のバケットのみに絞るのがベストですが、練習なのでFullAccessで進む
1. ステップ3:
    * 名前に「LambdaResizeRole」を入力し
    * 「ロールを作成」を押下

## 4.4 Lambda関数の作成
1. AWSコンソールで Lambda を開く -> 関数の作成
1. 「一から作成」を選択
1. 関数名: ImageResizerFunction
1. ランタイム: Python 3.12
1. アーキテクチャ: x86_64
1. 「デフォルトの実行ロールを変更」を展開 -> 既存のロールを使用する -> 先ほど作った LambdaResizeRole を選択
1. 「関数の作成」をクリック

## 4.5 コードの作成とライブラリの準備（ローカル作業）
1. CloudShellを開く
2. 以下のコマンドを順番に実行してください
```python
# 1. 作業フォルダへ移動
cd deploy_work

# 2. Python 3.12 用のライブラリをインストール
pip install Pillow -t . --platform manylinux2014_x86_64 --only-binary=:all: --python-version 3.12 --upgrade

# 3. 圧縮
zip -r deploy.zip .

# 4. 更新
aws lambda update-function-code --function-name ImageResizerFunction --zip-file fileb://deploy.zip
hello()
```

3. 結果を確認する
コマンドの最後に表示されるJSONで、以下のようになっているか確認してください。  
```python
"Runtime": "python3.12"
```

## 4.6 トリガーの設定（S3 → Lambda）
1. Lambda画面の上部にある図（関数の概要）の左側にある 「+ トリガーを追加」 をクリック
1. ソースを選択: プルダウンから S3 を選択
1. バケット: 作った 「入力用バケット」（...-input の方）を選択
1. イベントタイプ: 「すべてのオブジェクト作成イベント」を選択
1. サフィックス: .jpg と入力
1. 「私は、入力と出力の両方に同じ S3 バケットを使用することは推奨されておらず、この設定により、再帰呼び出しが生じ、それに伴い Lambda の使用量増加、コスト増大の可能性があることを了承します。」にチェックを入れる
1. 右下の 「追加」 ボタンをクリック

## 4.7 テスト
1. AWSコンソール で S3の画面を別タブで開きます
1. 「入力用バケット（input）」 を開きます
1. 適当な 画像ファイル（.jpg） をアップロードします
1. アップロード完了後、「出力用バケット（output）」 の方に移動して中身を確認します

## 4.8 実行失敗（トラブルシューティング）
前提：  
最初は、開発環境（Mac）と実行環境（AWS Linux）の差異に起因するトラブルがあった。そして、Lambda関数作成の際に、「ランタイム: Python 3.14」だったので、開発中（または超最新）のバージョンで、ライブラリ（Pillow）側がまだ正式に対応していないか、CloudShellがインストールしたバージョン（恐らく3.9か3.11）とバージョンの不一致を起こしていた。
CloudWatchのエラーメッセージ：  
```python
[ERROR] Runtime.ImportModuleError: Unable to import module 'lambda_function': cannot import name '_imaging' from 'PIL' (/var/task/PIL/__init__.py)

Traceback (most recent call last):

REPORT RequestId: aaad7136-102a-43e6-9f43-bdbfea6baf17 Duration: 301.13 ms Billed Duration: 302 ms Memory Size: 128 MB Max Memory Used: 71 MB Status: error Error Type: Runtime.ImportModuleError
``` 
🕵️‍♂️ 原因：Mac語とLinux語の違い/Pythonバージョンの問題  

解決策：  
    Lambda側の設定を 3.12 に戻す  
    1. AWSコンソールで Lambda の関数 ImageResizerFunction を開きます。  
    2. 「コード」 タブの下にある 「ランタイム設定」 の枠を見つけて、右の 「編集」 をクリック。  
    3. ランタイムを Python 3.12 に変更して保存します。

## 4.9 質問
* Q:料金について、かかりますか？  
A:今回の練習範囲なら、ほぼ間違いなく0円です。
    * Lambda: 月に100万回のリクエスト、40万GB秒の実行時間が無料（永久無料枠）。今回数回テストしただけなので 0円。
    * S3: 最初の12ヶ月は5GBまで無料。画像数枚なら数MBなので 0円。
    * CloudWatch Logs: 5GBまで無料。テキストログだけなら 0円。