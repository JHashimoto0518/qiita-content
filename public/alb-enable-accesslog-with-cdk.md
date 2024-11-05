---
title: AWS CDKでALBのアクセスログを有効にする
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

## CMKをサポートしていない

```bash
Error: Encryption key detected. Bucket encryption using KMS keys is unsupported
```

[Enable access logs for your Application Load Balancer - Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html)

> The bucket uses an unsupported server-side encryption option. The bucket must use Amazon S3-managed keys (SSE-S3).

## スタックのリージョン指定がない

```bash
Error: Region is required to enable ELBv2 access logging
```

https://www.perplexity.ai/search/error-region-is-required-to-en-dSQP_.YaRBGOqFMdgAOEew

https://github.com/aws/aws-cdk/issues/25007

- [ ] 転記する
