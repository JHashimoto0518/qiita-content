---
title: AWS CloudShellでの対話的操作で手軽にboto3を試すには
tags:
  - Python
  - AWS
  - 認証
  - boto3
  - CloudShell
private: false
updated_at: '2023-09-23T09:20:05+09:00'
id: 39fedf3ca5493f77aace
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

この記事では、AWS CloudShellでPythonのSDK、boto3を対話的に使用する方法を解説します。

CloudShellには、Pythonとboto3がプリインストールされています。さらに、CloudShellはマネージメントコンソールから認証情報を引き継ぐため、boto3に認証情報を明示的に渡す必要がありません。これにより、セットアップが不要です。

また、この手法は、AWSリソースの操作だけでなく、boto3のAPIを試す際にも非常に便利です。

# AWS CloudShellとは

AWS CloudShellは、マネージメントコンソールから直接アクセスできる、ブラウザベースのシェル環境です。AWS CLIや各種SDK、その他のシェルユーティリティをプリインストールされた状態で使用することができます。

https://docs.aws.amazon.com/ja_jp/cloudshell/latest/userguide/welcome.html

# boto3とは

boto3は、AWS SDKのPython版であり、AWSのリソースにプログラムからアクセスするためのPythonライブラリです。

https://aws.amazon.com/jp/sdk-for-python/

# boto3を対話的に実行する

AWS CloudShellを使用してboto3を対話的に実行する方法は、以下の通りです。

1. CloudShellの起動

    マネージメントコンソールにログイン後、ヘッダにあるCloudShellのアイコン（`>.`の形をしています）をクリックします。

2. Python インタープリタ起動
    シェルが起動したら、Python インタープリタを起動します。
    ```bash
    python3
    ```

    インタープリタを起動すると、`>>>` というプロンプトが表示されます。出力例を示します。

    ```bash:Output
    [cloudshell-user@ip-xx-x-xx-xxx ~]$ python3
    Python 3.7.16 (default, Aug 30 2023, 20:37:53) 
    [GCC 7.3.1 20180712 (Red Hat 7.3.1-15)] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>> 
    ```

    インタープリタを終了するまでは、Pythonの各種機能とboto3のAPIを対話的に試すことができます。以降のコマンドは、このインタープリタに対して実行します。

    尚、インタープリタは、タブで入力補完が効きます。

3. boto3インポート
    以下のコマンドを実行してboto3をインポートします。
    ```python
    import boto3
    ```

4. セッション初期化
    boto3のセッションを初期化します。認証情報とリージョンはマネジメントコンソールから引き継がれますが、明示的に指定することも可能です。

    東京リージョンを指定する例です。

    ```python
    sess = boto3.Session(region_name='ap-northeast-1')
    ```

    セッション初期化時に認証情報を渡すことも可能ですが、本記事では割愛します。

5. AWSリソース操作
    セッションが確立されたら、AWSの各種リソースを操作できます。

    セキュリティグループを一覧表示する例です。

    ```python
    ec2 = sess.client('ec2')
    res = ec2.describe_security_groups()
    print(res)
    ```

    ```python:Output
    {'SecurityGroups': [{'Description': 'for test', 'GroupName': 'test-sg', 'IpPermissions': [], ... 
    ```

    このようにJSONの出力が見づらい場合は、Pythonのjsonモジュールを使って整形することができます。具体的には、`json.dumps()`関数の`indent`引数を利用すると、きれいに整形された形で出力できます。

    ```python
    import json
    print(json.dumps(res, indent=2))
    ```

    ```python:Output
    {
      "SecurityGroups": [
        {
          "Description": "for test",
          "GroupName": "test-sg",
          "IpPermissions": [],
          "OwnerId": "3328xxxxxxxx",
          "GroupId": "sg-xxxxxxxxxxxxxxxxx",
          "IpPermissionsEgress": [
            {
              "IpProtocol": "-1",
              "IpRanges": [
                {
                  "CidrIp": "172.24.1.1/32",
                  "Description": "allow 172.24.1.1/32"
                }
              ],
              "Ipv6Ranges": [],
              "PrefixListIds": [],
              "UserIdGroupPairs": []
            }
          ],
          "VpcId": "vpc-xxxxxxxxxxxxxxxxx"
        },
        ...
    ```

6. インタープリタ終了
    最後にインタープリタを終了します。
    ```python
    quit()
    ```

# まとめ

この記事では、AWS CloudShellとboto3を用いて、AWSリソースを対話的に操作する方法について解説しました。

CloudShellであれば、セットアップなしでboto3を利用することが可能です。また、Pythonのインタープリタを使えば、対話的にboto3のAPIを試すことができます。
