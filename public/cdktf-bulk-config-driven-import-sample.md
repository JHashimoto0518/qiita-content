<!--
title: CDKTFのconfig-driven-importでS3バケットを一括インポートしてみた
tags:  AWS IaC Terraform CDKTF Import
-->


---
title: CDKTFのconfig-driven-importでS3バケットを一括インポートしてみた
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

# stash

- [ ] バケットの構成を変える

# はじめに

CDKTFでconfig-driven-importがサポートされました。

https://www.hashicorp.com/blog/cdktf-0-19-adds-support-for-config-driven-import-and-refactoring

クラスメソッドさんが、さっそく記事を公開されています。

https://dev.classmethod.jp/articles/cdktf-config-driven-import/

AWS SDKでリソースを列挙するコードを書けば、一括インポートが可能なのではないか、と思い試してみることにしまいた。

# CDKTFとは

# config-driven-importとは


# 複数バケットを用意する

まず、複数バケットを同じ構成で作成します。

```bash
export AWS_PROFILE=<profile name>

# Webサイトホスティング用バケット
BUCKET_NAME="cdktf-test-web-20231024"
aws s3api create-bucket --bucket $BUCKET_NAME --create-bucket-configuration LocationConstraint=ap-northeast-1

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

# 複数のバケットを一括インポートする（バケット名ハードコード）

バケットを一括でインポートしてみます。

まず、最も簡単と思われる、バケット名をハードコードする方法で実装します。

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

    // バケットのリスト
    const bucketList = [
      "cdktf-test-web-20231024",
      "cdktf-test-log-20231024",
    ];

    // すべてのバケットに対してインポートを実行
    for (const bucketName of bucketList) {
      new S3Bucket(this, bucketName, {}).importFrom(bucketName);
    }
  }
}

const app = new App();
new MyStack(app, "config-driven-import");
app.synth();
```

`cdktf diff`を実行すると、期待通り2つのバケットをインポートするplanが出力されます。

<details><summary>output</summary>

```bash
$ cdktf diff
config-driven-import  Initializing the backend...
config-driven-import  Initializing provider plugins...
config-driven-import  - Reusing previous version of hashicorp/aws from the dependency lock file
config-driven-import  - Using previously-installed hashicorp/aws v5.22.0

                      Terraform has been successfully initialized!
config-driven-import  
                      You may now begin working with Terraform. Try running "terraform plan" to see
                      any changes that are required for your infrastructure. All Terraform commands
                      should now work.

                      If you ever set or change modules or backend configuration for Terraform,
                      rerun this command to reinitialize your working directory. If you forget, other
                      commands will detect it and remind you to do so if necessary.
config-driven-import  aws_s3_bucket.cdktf-test-log-20231024 (cdktf-test-log-20231024): Preparing import... [id=cdktf-test-log-20231024]
config-driven-import  aws_s3_bucket.cdktf-test-web-20231024 (cdktf-test-web-20231024): Preparing import... [id=cdktf-test-web-20231024]
                      aws_s3_bucket.cdktf-test-log-20231024 (cdktf-test-log-20231024): Refreshing state... [id=cdktf-test-log-20231024]
                      aws_s3_bucket.cdktf-test-web-20231024 (cdktf-test-web-20231024): Refreshing state... [id=cdktf-test-web-20231024]
config-driven-import  Terraform will perform the following actions:
config-driven-import    # aws_s3_bucket.cdktf-test-log-20231024 (cdktf-test-log-20231024) will be imported
                          resource "aws_s3_bucket" "cdktf-test-log-20231024" {
                              arn                         = "arn:aws:s3:::cdktf-test-log-20231024"
                              bucket                      = "cdktf-test-log-20231024"
                              bucket_domain_name          = "cdktf-test-log-20231024.s3.amazonaws.com"
                              bucket_regional_domain_name = "cdktf-test-log-20231024.s3.ap-northeast-1.amazonaws.com"
                              hosted_zone_id              = "Z2M4EHUR26P7ZW"
                              id                          = "cdktf-test-log-20231024"
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

                        # aws_s3_bucket.cdktf-test-web-20231024 (cdktf-test-web-20231024) will be imported
                          resource "aws_s3_bucket" "cdktf-test-web-20231024" {
                              arn                         = "arn:aws:s3:::cdktf-test-web-20231024"
                              bucket                      = "cdktf-test-web-20231024"
                              bucket_domain_name          = "cdktf-test-web-20231024.s3.amazonaws.com"
                              bucket_regional_domain_name = "cdktf-test-web-20231024.s3.ap-northeast-1.amazonaws.com"
                              hosted_zone_id              = "Z2M4EHUR26P7ZW"
                              id                          = "cdktf-test-web-20231024"
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

                      Plan: 2 to import, 0 to add, 0 to change, 0 to destroy.
                      
                      ─────────────────────────────────────────────────────────────────────────────

                      Saved the plan to: plan

                      To perform exactly these actions, run the following command to apply:
                          terraform apply "plan

```

</details>

`cdktf deploy`で、インポートを実行します。

<details><summary>output</summary>

```bash
$ cdktf deploy
config-driven-import  Initializing the backend...
config-driven-import  Initializing provider plugins...
config-driven-import  - Reusing previous version of hashicorp/aws from the dependency lock file
config-driven-import  - Using previously-installed hashicorp/aws v5.22.0
config-driven-import  Terraform has been successfully initialized!
                      
                      You may now begin working with Terraform. Try running "terraform plan" to see
                      any changes that are required for your infrastructure. All Terraform commands
                      should now work.

                      If you ever set or change modules or backend configuration for Terraform,
                      rerun this command to reinitialize your working directory. If you forget, other
                      commands will detect it and remind you to do so if necessary.
config-driven-import  aws_s3_bucket.cdktf-test-log-20231024 (cdktf-test-log-20231024): Preparing import... [id=cdktf-test-log-20231024]
config-driven-import  aws_s3_bucket.cdktf-test-web-20231024 (cdktf-test-web-20231024): Preparing import... [id=cdktf-test-web-20231024]
                      aws_s3_bucket.cdktf-test-web-20231024 (cdktf-test-web-20231024): Refreshing state... [id=cdktf-test-web-20231024]
                      aws_s3_bucket.cdktf-test-log-20231024 (cdktf-test-log-20231024): Refreshing state... [id=cdktf-test-log-20231024]
config-driven-import  Terraform will perform the following actions:
config-driven-import    # aws_s3_bucket.cdktf-test-log-20231024 (cdktf-test-log-20231024) will be imported
                          resource "aws_s3_bucket" "cdktf-test-log-20231024" {
                              arn                         = "arn:aws:s3:::cdktf-test-log-20231024"
                              bucket                      = "cdktf-test-log-20231024"
                              bucket_domain_name          = "cdktf-test-log-20231024.s3.amazonaws.com"
                              bucket_regional_domain_name = "cdktf-test-log-20231024.s3.ap-northeast-1.amazonaws.com"
                              hosted_zone_id              = "Z2M4EHUR26P7ZW"
                              id                          = "cdktf-test-log-20231024"
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

                        # aws_s3_bucket.cdktf-test-web-20231024 (cdktf-test-web-20231024) will be imported
                          resource "aws_s3_bucket" "cdktf-test-web-20231024" {
                              arn                         = "arn:aws:s3:::cdktf-test-web-20231024"
                              bucket                      = "cdktf-test-web-20231024"
                              bucket_domain_name          = "cdktf-test-web-20231024.s3.amazonaws.com"
                              bucket_regional_domain_name = "cdktf-test-web-20231024.s3.ap-northeast-1.amazonaws.com"
                              hosted_zone_id              = "Z2M4EHUR26P7ZW"
                              id                          = "cdktf-test-web-20231024"
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

                      Plan: 2 to import, 0 to add, 0 to change, 0 to destroy.
                      
                      Do you want to perform these actions?
                        Terraform will perform the actions described above.
                        Only 'yes' will be accepted to approve.
config-driven-import  Enter a value: yes
config-driven-import
config-driven-import  aws_s3_bucket.cdktf-test-log-20231024 (cdktf-test-log-20231024): Importing... [id=cdktf-test-log-20231024]
                      aws_s3_bucket.cdktf-test-log-20231024 (cdktf-test-log-20231024): Import complete [id=cdktf-test-log-20231024]
                      aws_s3_bucket.cdktf-test-web-20231024 (cdktf-test-web-20231024): Importing... [id=cdktf-test-web-20231024]
                      aws_s3_bucket.cdktf-test-web-20231024 (cdktf-test-web-20231024): Import complete [id=cdktf-test-web-20231024]
                      
                      Apply complete! Resources: 2 imported, 0 added, 0 changed, 0 destroyed.

No outputs found.
```

</details>

インポートが完了したら、`importFrom`の呼び出しは不要なので、削除します。

```diff_typescript:main.ts
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

    // バケットのリスト
    const bucketList = [
      "cdktf-test-web-20231024",
      "cdktf-test-log-20231024",
    ];

    // すべてのバケットに対してインポートを実行
    for (const bucketName of bucketList) {
-     new S3Bucket(this, bucketName, {}).importFrom(bucketName);        
+     new S3Bucket(this, bucketName, {});
    }
  }
}

const app = new App();
new MyStack(app, "config-driven-import");
app.synth();
```

コードを変更したので、念のため、再度`cdktf diff`を実行します。

<details><summary>output</summary>

```bash
$ cdktf diff
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
config-driven-import  aws_s3_bucket.cdktf-test-web-20231024 (cdktf-test-web-20231024): Refreshing state... [id=cdktf-test-web-20231024]
config-driven-import  aws_s3_bucket.cdktf-test-log-20231024 (cdktf-test-log-20231024): Refreshing state... [id=cdktf-test-log-20231024]
config-driven-import  No changes. Your infrastructure matches the configuration.

                      
config-driven-import  Terraform has compared your real infrastructure against your configuration
                      and found no differences, so no changes are needed.
```

</details>

期待通りに`No changes.`が出力され、コードとバケットの同期がとれていることがわかります。



# 複数バケットを一括インポートする (AWS SDK使用)



# スナップショットテストによるガード


# リファクタリング



## AWS SDKのインストール

次に、動的にバケットを列挙するため、AWS SDKをインストールします。

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

AWSプロバイダーに資格情報を渡す方法は複数ありますが、今回は、プロファイルと環境変数AWS_PROFILEを使用します。

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

Terraformの構成ファイルと実際のバケットの状態に差異がないか、確認します。

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
