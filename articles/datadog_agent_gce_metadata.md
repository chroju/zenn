---
title: "Datadog AgentはデフォルトでGCEのメタデータを得ようとする"
emoji: "🐾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Datadog"]
published: true
publication_name: "globis"
---

先日偶然見つけたのですが、Datadog Agentがこんなログを出していました。

```
2024-07-10 09:32:13 UTC | CORE | WARN | (pkg/util/cloudproviders/gce/gce_tags.go:49 in getCachedTags) | unable to get tags from gce and cache is empty: GCE metadata API error: status code 401 trying to GET http://169.254.169.254/computeMetadata/v1/?recursive=true
2024-07-10 09:12:54 UTC | CORE | INFO | (comp/metadata/host/utils/tags.go:158 in GetHostTags) | Unable to get host tags from source: gce - using cached host tags
```

内容としては難しいものではなく、GCEのVMメタデータやタグ情報をうまく取得できていないということのようです。

問題は、このDatadog Agentは **Amazon EKSのクラスタ上で動作している** ということです。

## どういうことだってばよ

念のため確認しましたが、EKS (EC2)としてのメタデータ取得は正常に行えていますし、DatadogのAWS Integrationも問題なく設定されていました。

それなのに、なぜGCEのメタデータを取得しようとするのかと言えば、単純な話でDatadog Agentのデフォルト設定がそうなっているからでした。

```yaml
## @param cloud_provider_metadata - list of strings -  optional - default: ["aws", "gcp", "azure", "alibaba", "oracle", "ibm"]
## @env DD_CLOUD_PROVIDER_METADATA - space separated list of strings - optional - default: aws gcp azure alibaba oracle ibm
...
## @param collect_gce_tags - boolean - optional - default: true
## @env DD_COLLECT_GCE_TAGS - boolean - optional - default: true
## Collect Google Cloud Engine metadata as host tags
#
# collect_gce_tags: true
```

ソースとしては [こちら](https://github.com/DataDog/datadog-agent/blob/78ca7aad3407f9202fc0ae3f81623907a9e32691/pkg/config/config_template.yaml#L345-L422) になります。

`cloud_provider_metadata` というパラメータは、メタデータ取得を行うCloud Providerの設定で、デフォルトだとAWSとGCPのみならず、Azure、Alibaba、Oracle、IBMのメタデータも取得しようと試みるようです。安全側に倒されたデフォルト値という感じですね。

面白いのが `collect_gce_tags` で、これが `default: true` になっています。似たパラメータの `collect_ec2_tags` は `default: false` なので不思議な感じがしますね。

ということで、この2つのパラメータを適切に設定すれば、冒頭のログは表示されなくなりました。このログはERRORレベルではありませんし、出しっぱなしでも明確に困るということはないと思います。が、いずれのログも30分おきに出力されるため、Agentを多数動作させている環境だと、それなりにコストへ跳ね返ってくるかもしれません。
