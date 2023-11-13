<!--
title: CDKTFのconfig-driven-importでS3バケットを一括インポートしてみた
tags:  AWS IaC Terraform CDKTF Import
-->


---
title: CDKTF の config-driven-import で S3 バケットを一括インポートしてみた
tags:
  - 'AWS'
  - 'IaC'
  - 'Terraform'
  - 'CDKTF'
  - 'Import'
private: false
updated_at: ''
id: cdktf-bulk-config-driven-import-sample
organization_url_name: null
slide: false
ignorePublish: false
---

# 調査

## AwsSdkCallでListBucketsを呼び出せないか

- [javascript - Correct way to use AWS SDK within AWS CDK - Stack Overflow](https://stackoverflow.com/questions/59406959/correct-way-to-use-aws-sdk-within-aws-cdk)
- [interface AwsSdkCall · AWS CDK](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.custom_resources.AwsSdkCall.html)
- [AwsSdkCall インターフェイスを使用して SDK 呼び出しを行う | AWS re:Post](https://repost.aws/ja/knowledge-center/cdk-sdk-calls-awssdkcall)
- [[AWS CDK] APIを呼び出すだけのカスタムリソースならLambda関数は不要な件 | DevelopersIO](https://dev.classmethod.jp/articles/create-custom-resources-with-aws-cdk-without-using-lambda-functions/)
- [AWS CDKで別リージョンにスタックをデプロイしてパラメータをリージョン間で受け渡す方法 －AWS CDKカスタムリソースの実装例 - NRIネットコムBlog](https://tech.nri-net.com/entry/aws_cdk_cross_region_stack_deployment_method)

### CloudFormationのカスタムリソースとは

- [AWS CloudFormationのカスタムリソースでRDSやElasticsearchをアップデートする仕組みを作る - Cybozu Inside Out | サイボウズエンジニアのブログ](https://blog.cybozu.io/entry/2019/06/19/080000)
- [CloudFormationカスタムリソースを学ぶ](https://zenn.dev/dehio3/articles/f449b3ed652aad)

## Terraformのインポートブロックで実現できないか

- [Terraform 1.5 で追加される import ブロックの使い方](https://zenn.dev/kou_pg_0131/articles/tf-import-block)
- [Terraform 1.6 adds a test framework for enhanced code validation](https://www.hashicorp.com/blog/terraform-1-6-adds-a-test-framework-for-enhanced-code-validation)
- [Import - Configuration Language | Terraform | HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import#examples)

## CDKのConstructをCDKTFから呼び出す

https://dev.classmethod.jp/articles/cdk-for-terraform-aws-adapter-aws-cdk-construct/

## どうやればStackにバケット名リストを渡せるか？

- [cdktfドキュメント翻訳](https://zenn.dev/uta_mory/scraps/98d7236c185dde)
- [CDK for TerraformでGoogle Cloudのリソースを作ってみた 発動篇](https://zenn.dev/cloud_ace/articles/cdk-for-terraform-functions)

---

# はじめに

CDKTF で config-driven-import がサポートされました。

https://www.hashicorp.com/blog/cdktf-0-19-adds-support-for-config-driven-import-and-refactoring

クラスメソッドさんが、さっそく記事を公開されています。

https://dev.classmethod.jp/articles/cdktf-config-driven-import/

AWS SDK でリソースを列挙するコードを書けば、一括インポートが可能なのではないか、と思い試してみることにしまいた。

# CDKTFとは

https://developer.hashicorp.com/terraform/cdktf

# config-driven-importとは


# 複数バケットを用意する

まず、複数バケットを作成します。Web サイト用バケットのみバージョニングを有効化します。

```bash
export AWS_PROFILE=<profile name>

# Webサイト用バケット
BUCKET_NAME="cdktf-test-web-20231024"
aws s3api create-bucket --bucket $BUCKET_NAME --create-bucket-configuration LocationConstraint=ap-northeast-1

# バージョンニングを有効化
aws s3api put-bucket-versioning --bucket $BUCKET_NAME --versioning-configuration Status=Enabled

# ログ用バケット
BUCKET_NAME="cdktf-test-log-20231024"
aws s3api create-bucket --bucket $BUCKET_NAME --create-bucket-configuration LocationConstraint=ap-northeast-1
```

```bash:output
{
    "Location": "http://cdktf-test-web-20231024.s3.amazonaws.com/"
}
{
    "Location": "http://cdktf-test-log-20231024.s3.amazonaws.com/"
}
```

# CDKTFプロジェクトを初期化する

プロジェクトを初期化します。

```bash
mkdir bulk-import-sample
cd bulk-import-sample
cdktf init --template="typescript" --providers="aws@~>5.0" --local
```

解説については、こちらをご参照ください。

https://qiita.com/JHashimoto/items/81c938d1a0574f1fa34c

# 複数のバケットを一括インポートする（Data Source使用）

複数バケットをサポートする Data Source はない。

https://stackoverflow.com/questions/72483512/terraform-data-block-all-buckets

単一バケットのみサポートしている。

https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/s3_bucket?lang=typescript

# 複数のバケットを一括インポートする（Input Variableの値をハードコードで外部から渡す）

インポートではなく、Create する plan が出力される。

```bash
cdktf diff --var 'bucket_names=["cdktf-tes
t-web-20231024","cdktf-test-log-20231024"]'
...
                      Terraform will perform the following actions:
config-driven-import    # aws_s3_bucket.bucket (bucket)["cdktf-test-log-20231024"] will be created
...
                      Plan: 2 to add, 0 to change, 0 to destroy.
...
```

インポートします。

```bash
cdktf deploy --var 'bucket_names=["cdktf-t
est-web-20231024","cdktf-test-log-20231024"]'
...
                      Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

No outputs found.
```

# 複数のバケットを一括インポートする（Input Variableの値をAWS CLIで外部から渡す）

インポートではなく、Create する plan が出力される。

```bash
BUCKET_NAMES_JSON=$(aws s3api list-buckets --query "Buckets[?starts_with(Name,'cdktf-test-')].Name" --output json)
echo $BUCKET_NAMES_JSON 
# [ "cdktf-test-log-20231024", "cdktf-test-web-20231024" ]
cdktf diff --var "bucket_names=$BUCKET_NAMES_JSON"
```

## 参考

https://github.com/jcolemorrison/ecs-microservices-cdktf/tree/main

# 複数のバケットを一括インポートする（AWS SDK使用）

バケットを一括でインポートしてみます。

- [ ] `cdktf diff`で`no changes.`が出力される。レスポンスの前にコンストラクタのスコープを抜けていないか？
- [ ] AWSSDKCall を試してみる
  * [AwsSdkCall インターフェイスを使用した SDK 呼び出しの実行 |AWS re:Post](https://repost.aws/knowledge-center/cdk-sdk-calls-awssdkcall)
  * [javascript - AWS CDK 内で AWS SDK を使用する正しい方法 - スタックオーバーフロー](https://stackoverflow.com/questions/59406959/correct-way-to-use-aws-sdk-within-aws-cdk)
  * [インターフェイス AwsSdkCall ·AWS CDK](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.custom_resources.AwsSdkCall.html)

```ts:main.ts
import { Construct } from "constructs";
import { App, TerraformStack } from "cdktf";
import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
import { S3Bucket } from "@cdktf/provider-aws/lib/s3-bucket";
import { S3Client, ListBucketsCommand } from "@aws-sdk/client-s3";

class MyStack extends TerraformStack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new AwsProvider(this, "AWS", {
      region: "ap-northeast-1",
    });

    const importBuckets = async () => {
      const client = new S3Client();
      const input = {};
      const command = new ListBucketsCommand(input);
      const response = await client.send(command);

      if (response && response.Buckets) {
        // すべてのバケットに対してインポートを実行
        for (const bucket of response.Buckets) {
          const name = bucket.Name!;
          new S3Bucket(this, name, {}).importFrom(name);
        }
      }
    };

    importBuckets();
  }
}

const app = new App();
new MyStack(app, "config-driven-import");
app.synth();
```

## 参考

https://go-tech.blog/nodejs/ts-aws-sdk-s3/

https://dev.classmethod.jp/articles/pre-signed-cdk-sdk-s3-api/#toc-13

# AWS SDKの認証

* [AWS SDK for JavaScript v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-client-s3/Interface/Bucket/)
* [@aws-sdk/credential-providers | AWS SDK for JavaScript v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/modules/_aws_sdk_credential_providers.html)
* [AWS SDK v3 における認証](https://zenn.dev/luma/articles/bd3c59b3d7682d)
* [AWS SDK for JavaScript](https://aws.amazon.com/jp/sdk-for-javascript/)
* [@aws-sdk/credential-providers - npm](https://www.npmjs.com/package/@aws-sdk/credential-providers#fromsso)
* [Configuration and authentication settings reference - AWS SDKs and Tools](https://docs.aws.amazon.com/sdkref/latest/guide/settings-reference.html#EVarSettings)

# リファクタリング

https://developer.hashicorp.com/terraform/cdktf/test/unit-tests
 
https://zenn.dev/uta_mory/scraps/98d7236c185dde


# スナップショットテストによるガード


## AWS SDKのインストール

次に、動的にバケットを列挙するため、AWS SDK をインストールします。

```bash
npm install @aws-sdk/client-s3
```

https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/s3/

## 任意数のバケットインポートに対応


# バケットをCDKTFで構成変更する


----

# 単一のバケットをインポートする（テスト）

```bash
export AWS_PROFILE=<profile name>
BUCKET_NAME="test-cdktf-import-20231022"
ws s3api create-bucket --bucket $BUCKET_NAME --create-bucket-configuration LocationConstraint=ap-northeast-1
```

```bash
mkdir single-import-sample
cd single-import-sample/
cdktf init --template="typescript" --providers="aws@~>5.0" --local
```

## AWS資格情報の設定

AWS プロバイダーに資格情報を渡す方法は複数ありますが、今回は、プロファイルと環境変数 AWS_PROFILE を使用します。

```bash
export AWS_PROFILE=<profile name>
```

## CDKコードの編集

`main.ts`を編集します。

```ts:main.ts
import { Construct } from "constructs";
import { App, TerraformStack } from "cdktf";
import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
import { S3Bucket } from "@cdktf/provider-aws/lib/s3-bucket";

class MyStack extends TerraformStack {
  constructor(scope: Construct, id: string) {
    super(scope, id);
    new AwsProvider(this, "AWS", {
      region: "ap-northeast-1",
    });

    new S3Bucket(this, "bucket", {}).importFrom("test-cdktf-import-20231022");
  }
}

const app = new App();
new MyStack(app, "config-driven-import");
app.synth();
```

```bash
cdktf diff
```

<details><summary>output</summary>

```bash
config-driven-import  Initializing the backend...
config-driven-import  
                      Successfully configured the backend "local"! Terraform will automatically
                      use this backend unless the backend configuration changes.
config-driven-import  Initializing provider plugins...
                      - Finding hashicorp/aws versions matching "5.22.0"...
config-driven-import  - Installing hashicorp/aws v5.22.0...
config-driven-import  - Installed hashicorp/aws v5.22.0 (signed by HashiCorp)
config-driven-import  Terraform has created a lock file .terraform.lock.hcl to record the provider
                      selections it made above. Include this file in your version control repository
                      so that Terraform can guarantee to make the same selections by default when
                      you run "terraform init" in the future.

                      Terraform has been successfully initialized!
                      
                      You may now begin working with Terraform. Try running "terraform plan" to see
                      any changes that are required for your infrastructure. All Terraform commands
                      should now work.

                      If you ever set or change modules or backend configuration for Terraform,
                      rerun this command to reinitialize your working directory. If you forget, other
                      commands will detect it and remind you to do so if necessary.
config-driven-import  - Fetching hashicorp/aws 5.22.0 for linux_amd64...
config-driven-import  - Retrieved hashicorp/aws 5.22.0 for linux_amd64 (signed by HashiCorp)
                      - Obtained hashicorp/aws checksums for linux_amd64; All checksums for this platform were already tracked in the lock file
config-driven-import  Success! Terraform has validated the lock file and found no need for changes.
config-driven-import  aws_s3_bucket.bucket (bucket): Preparing import... [id=test-cdktf-import-20231022]
config-driven-import  aws_s3_bucket.bucket (bucket): Refreshing state... [id=test-cdktf-import-20231022]
config-driven-import  Terraform will perform the following actions:
config-driven-import    # aws_s3_bucket.bucket (bucket) will be imported
                          resource "aws_s3_bucket" "bucket" {
                              arn                         = "arn:aws:s3:::test-cdktf-import-20231022"
                              bucket                      = "test-cdktf-import-20231022"
                              bucket_domain_name          = "test-cdktf-import-20231022.s3.amazonaws.com"
                              bucket_regional_domain_name = "test-cdktf-import-20231022.s3.ap-northeast-1.amazonaws.com"
                              hosted_zone_id              = "Z2M4EHUR26P7ZW"
                              id                          = "test-cdktf-import-20231022"
                              object_lock_enabled         = false
                              region                      = "ap-northeast-1"
                              request_payer               = "BucketOwner"
                              tags                        = {}
                              tags_all                    = {}

                              grant {
                                  id          = "d03aad499a43b0490edc04b16d5e8673281a97f4127f8b0a8f2a3d6ff57c0598"
                                  permissions = [
                                      "FULL_CONTROL",
                                  ]
                                  type        = "CanonicalUser"
                              }

                              server_side_encryption_configuration {
                                  rule {
                                      bucket_key_enabled = false

                                      apply_server_side_encryption_by_default {
                                          sse_algorithm = "AES256"
                                      }
                                  }
                              }

                              versioning {
                                  enabled    = false
                                  mfa_delete = false
                              }
                          }

                      Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
                      
                      ─────────────────────────────────────────────────────────────────────────────

                      Saved the plan to: plan

                      To perform exactly these actions, run the following command to apply:
                          terraform apply "plan"
```

</details>

インポートを実行します。

```bash
cdktf deploy
config-driven-import  Initializing the backend...
config-driven-import  Initializing provider plugins...
config-driven-import  - Reusing previous version of hashicorp/aws from the dependency lock file
config-driven-import  - Using previously-installed hashicorp/aws v5.22.0

                      Terraform has been successfully initialized!
                      
                      You may now begin working with Terraform. Try running "terraform plan" to see
                      any changes that are required for your infrastructure. All Terraform commands
                      should now work.

                      If you ever set or change modules or backend configuration for Terraform,
                      rerun this command to reinitialize your working directory. If you forget, other
                      commands will detect it and remind you to do so if necessary.
config-driven-import  aws_s3_bucket.bucket (bucket): Preparing import... [id=test-cdktf-import-20231022]
config-driven-import  aws_s3_bucket.bucket (bucket): Refreshing state... [id=test-cdktf-import-20231022]
config-driven-import  Terraform will perform the following actions:
config-driven-import    # aws_s3_bucket.bucket (bucket) will be imported
                          resource "aws_s3_bucket" "bucket" {
                              arn                         = "arn:aws:s3:::test-cdktf-import-20231022"
                              bucket                      = "test-cdktf-import-20231022"
                              bucket_domain_name          = "test-cdktf-import-20231022.s3.amazonaws.com"
                              bucket_regional_domain_name = "test-cdktf-import-20231022.s3.ap-northeast-1.amazonaws.com"
                              hosted_zone_id              = "Z2M4EHUR26P7ZW"
                              id                          = "test-cdktf-import-20231022"
                              object_lock_enabled         = false
                              region                      = "ap-northeast-1"
                              request_payer               = "BucketOwner"
                              tags                        = {}
                              tags_all                    = {}

                              grant {
                                  id          = "d03aad499a43b0490edc04b16d5e8673281a97f4127f8b0a8f2a3d6ff57c0598"
                                  permissions = [
                                      "FULL_CONTROL",
                                  ]
                                  type        = "CanonicalUser"
                              }

                              server_side_encryption_configuration {
                                  rule {
                                      bucket_key_enabled = false

                                      apply_server_side_encryption_by_default {
                                          sse_algorithm = "AES256"
                                      }
                                  }
                              }

                              versioning {
                                  enabled    = false
                                  mfa_delete = false
                              }
                          }

                      Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
config-driven-import  
                      Do you want to perform these actions?
                        Terraform will perform the actions described above.
                        Only 'yes' will be accepted to approve.

Please review the diff output above for config-driven-import
❯ Approve  Applies the changes outlined in the plan.
  Dismiss
  Stop
```

```bash
config-driven-import  Enter a value: yes

                      aws_s3_bucket.bucket (bucket): Importing... [id=test-cdktf-import-20231022]
                      aws_s3_bucket.bucket (bucket): Import complete [id=test-cdktf-import-20231022]
config-driven-import  
                      Apply complete! Resources: 1 imported, 0 added, 0 changed, 0 destroyed.

No outputs found.
```

Terraform の構成ファイルと実際のバケットの状態に差異がないか、確認します。

```bash
cdktf diff
config-driven-import  Initializing the backend...
config-driven-import  Initializing provider plugins...
                      - Reusing previous version of hashicorp/aws from the dependency lock file
config-driven-import  - Using previously-installed hashicorp/aws v5.22.0

                      Terraform has been successfully initialized!
                      
                      You may now begin working with Terraform. Try running "terraform plan" to see
                      any changes that are required for your infrastructure. All Terraform commands
                      should now work.

                      If you ever set or change modules or backend configuration for Terraform,
                      rerun this command to reinitialize your working directory. If you forget, other
                      commands will detect it and remind you to do so if necessary.
config-driven-import  aws_s3_bucket.bucket (bucket): Refreshing state... [id=test-cdktf-import-20231022]
config-driven-import  No changes. Your infrastructure matches the configuration.

                      
config-driven-import  Terraform has compared your real infrastructure against your configuration
                      and found no differences, so no changes are needed.
```

バージョニングを有効にします。

```bash
cdktf diff
config-driven-import  Initializing the backend...
config-driven-import  Initializing provider plugins...
                      - Reusing previous version of hashicorp/aws from the dependency lock file
config-driven-import  - Using previously-installed hashicorp/aws v5.22.0
config-driven-import  Terraform has been successfully initialized!
                      
                      You may now begin working with Terraform. Try running "terraform plan" to see
                      any changes that are required for your infrastructure. All Terraform commands
                      should now work.

                      If you ever set or change modules or backend configuration for Terraform,
                      rerun this command to reinitialize your working directory. If you forget, other
                      commands will detect it and remind you to do so if necessary.
config-driven-import  aws_s3_bucket.bucket (bucket): Refreshing state... [id=test-cdktf-import-20231022]
config-driven-import  Terraform used the selected providers to generate the following execution
                      plan. Resource actions are indicated with the following symbols:
                        + create

                      Terraform will perform the following actions:
config-driven-import    # aws_s3_bucket_versioning.versioning (versioning) will be created
                        + resource "aws_s3_bucket_versioning" "versioning" {
                            + bucket = "test-cdktf-import-20231022"
                            + id     = (known after apply)

                            + versioning_configuration {
                                + mfa_delete = (known after apply)
                                + status     = "Enabled"
                              }
                          }

                      Plan: 1 to add, 0 to change, 0 to destroy.
                      
                      ─────────────────────────────────────────────────────────────────────────────

                      Saved the plan to: plan

                      To perform exactly these actions, run the following command to apply:
                          terraform apply "plan"
```

変更を適用します。

```bash
cdktf deploy
config-driven-import  Initializing the backend...
config-driven-import  Initializing provider plugins...
                      - Reusing previous version of hashicorp/aws from the dependency lock file
config-driven-import  - Using previously-installed hashicorp/aws v5.22.0

                      Terraform has been successfully initialized!
                      
                      You may now begin working with Terraform. Try running "terraform plan" to see
                      any changes that are required for your infrastructure. All Terraform commands
                      should now work.

                      If you ever set or change modules or backend configuration for Terraform,
                      rerun this command to reinitialize your working directory. If you forget, other
                      commands will detect it and remind you to do so if necessary.
config-driven-import  aws_s3_bucket.bucket (bucket): Refreshing state... [id=test-cdktf-import-20231022]
config-driven-import  Terraform used the selected providers to generate the following execution plan.
                      Resource actions are indicated with the following symbols:
                        + create

                      Terraform will perform the following actions:

                        # aws_s3_bucket_versioning.versioning (versioning) will be created
                        + resource "aws_s3_bucket_versioning" "versioning" {
                            + bucket = "test-cdktf-import-20231022"
                            + id     = (known after apply)

                            + versioning_configuration {
                                + mfa_delete = (known after apply)
                                + status     = "Enabled"
                              }
                          }

                      Plan: 1 to add, 0 to change, 0 to destroy.
                      
                      Do you want to perform these actions?
                        Terraform will perform the actions described above.
                        Only 'yes' will be accepted to approve.

Please review the diff output above for config-driven-import
❯ Approve  Applies the changes outlined in the plan.
  Dismiss
  Stop
```

```bash
config-driven-import  Enter a value: yes
config-driven-import  aws_s3_bucket_versioning.versioning (versioning): Creating...
config-driven-import  aws_s3_bucket_versioning.versioning (versioning): Creation complete after 2s [id=test-cdktf-import-20231022]
config-driven-import  
                      Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

No outputs found.
```

差異がないことを確認します。

```bash
cdktf diff
config-driven-import  Initializing the backend...
config-driven-import  Initializing provider plugins...
                      - Reusing previous version of hashicorp/aws from the dependency lock file
config-driven-import  - Using previously-installed hashicorp/aws v5.22.0

                      Terraform has been successfully initialized!
config-driven-import  
                      You may now begin working with Terraform. Try running "terraform plan" to see
                      any changes that are required for your infrastructure. All Terraform commands
                      should now work.

                      If you ever set or change modules or backend configuration for Terraform,
                      rerun this command to reinitialize your working directory. If you forget, other
                      commands will detect it and remind you to do so if necessary.
config-driven-import  aws_s3_bucket.bucket (bucket): Refreshing state... [id=test-cdktf-import-20231022]
config-driven-import  aws_s3_bucket_versioning.versioning (versioning): Refreshing state... [id=test-cdktf-import-20231022]
config-driven-import  No changes. Your infrastructure matches the configuration.

                      Terraform has compared your real infrastructure against your configuration
                      and found no differences, so no changes are needed.
```
