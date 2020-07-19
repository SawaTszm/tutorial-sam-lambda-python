# lambdaチュートリアル

## 資料

* [Python による Lambda 関数のビルド - AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/lambda-python.html)
* [Python の AWS Lambda デプロイパッケージ - AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/python-package.html)

## 基本

1. Lambdaの関数をAWSコンソールで手動作成
2. ローカルでコードを作成・編集してZIPに固める(デプロイパッケージを作る)
3. LambdaAPIを利用してそのパッケージをアップロードする

### lambda関数の作成

IAMで基本的なロールを作成する。

* [信頼されたエンティティ] – [Lambda]
* [アクセス許可] – [AWSLambdaBasicExecutionRole]
* ロール名 – lambda-role

> AWSLambdaBasicExecutionRole ポリシーには、ログを CloudWatch Logs に書き込むために関数が必要とするアクセス許可があります。

それができたら、lambdaコンソールで関数を作成。

* Name - my-function
* [ランタイム] - [Python 3.8]
* [ロール] - [既存のロールを選択]
* 既存のロール - lambda-role

作成できたら、テストイベントを設定する。

* [テスト]を選択
* [イベント名]に任意の名前を入力
* [作成]を選択後、[テスト]を選択すると実行できる

lambdaコンソールでも簡単なコード編集は出来、セーブするたびにデプロイパッケージを自動で作成してくれている。  
しかし外部パッケージのインストールなどには対応していないため、boto3以外のものを使用する場合はデプロイパッケージをローカルで作成したのちアップデートする必要がある。  

> デプロイパッケージは、関数のコードと依存関係を含む ZIP アーカイブです。Lambda API を使用して関数を管理する場合や、AWS SDK 以外のライブラリや依存関係を含める必要がある場合は、デプロイパッケージを作成する必要があります。パッケージを Lambda に直接アップロードするか、Amazon S3 バケットを使用して Lambda にアップロードすることができます。デプロイパッケージが 50 MB 以上の場合は、Amazon S3 を使用する必要があります。

### ローカルでのコード作成

基本の設定では、lambda関数は`lambda_function.lambda_handler`をハンドラ値としている。  
そのため作成する際は以下のフォルダ構成になる。

```text
my-function
    L lambda_function.py
    L function.zip
    L package
        L PIL
        L ...
```

### デプロイパッケージ作成

`lambda_function.py`内でコードを修正・更新したら、デプロイパッケージ(ZIPアーカイブ)を作成する。

```bash
~my-function$ zip function.zip lambda_function.py
```

### デプロイパッケージのアップロード

それができたら、lambdaAPIでアップロードする。

```bash
~my-function$ aws lambda update-function-code --function-name my-function --zip-file fileb://function.zip
```

## 外部ライブラリをデプロイパッケージに含める場合

`pip`を使ってローカルディレクトリにインストールして、デプロイパッケージに含める。  

```bash
~/my-function$ pip install --target ./package Pillow
```

それができたら、依存関係のZIPアーカイブを作成。

```bash
~/my-function$ cd package

~/my-function/package$ zip -r9 ${OLDPWD}/function.zip .
```

関数コード(lambda_function.py)をアーカイブに追加。

```bash
~/my-function/package$ cd $OLDPWD

~/my-function$ zip -g function.zip lambda_function.py
```

デプロイパッケージををアップロード。

```bash
~/my-function$ aws lambda update-function-code --function-name my-function --zip-file fileb://function.zip
```

## 仮想環境がある場合

`virtualenv`, `venv`などでインストールする必要がある場合。  
(Python3.3以降では組み込みのvnevで仮想環境を作成可能)

仮想環境を作成&アクティブ化。

```bash
~/my-function$ python3 -m venv v-env

~/my-function$ source v-env/bin/activate
```

pipで仮想環境にライブラリをインストール。

```bash
(v-env) ~/my-function$ pip install Pillow
```

仮想環境を無効化。

```bash
(v-env) ~/my-function$ deactivate
```

ライブラリの内容でZIPアーカイブを作成。

```bash
~/my-function/v-env/lib/python3.8/site-packages$ zip -r9 ${OLDPWD}/function.zip .
```

依存関係は、ライブラリに応じて`site-packages`または`dist-packages`に表示され、仮想環境の最初のフォルダは`lib`または`lib64`になる。

関数コードをアーカイブに追加。

```bash
~/my-function/v-env/lib/python3.8/site-packages$ cd $ OLDPWD
~/my-function$ zip -g function.zip lambda_function.py
```

デプロイパッケージをアップロード。

```bash
~/my-function$ aws lambda update-function-code --function-name my-function --zip-file fileb://function.zip
```

## まとめ

lambda関数をコンソールから作成し、ロールを渡すところまでやってやれば、ローカルで開発したコードをlambdaAPIでいつでもアップロード出来る。  
が、外部パッケージを使用する場合はデプロイパッケージに全ソースを含めないといけないため、

* ローカルディレクトリに明示的にインストールして、ZIPアーカイブに含める
* 仮想環境でインストールして、そのライブラリ管理ディレクトリをZIPアーカイブに含める

のどちらかの対応が必須。  

正直かなり手順が面倒くさい上にデプロイ後にしか動作確認できないのがネックなので、lambda関数自体の設定・作成、ローカルでの動作確認、デプロイパッケージ作成→アップロードを出来るようにしたものがSAM、なんだと思う。
