---
title: "argocdでデフォルトではないネームスペースを使う場合はtracking labelを変更したほうがよさそう"
date: "2024-11-09T18:50:11+09:00"
draft: false
---

この記事を書こうと思って argocd のドキュメントを見てたら似たようなことを書いてた。(けど本質的には別問題)

> Switch resource tracking method¶
>
> Also, while technically not necessary, it is strongly suggested that you switch the application tracking method from the default label setting to either annotation or annotation+label. The reasoning for this is, that application names will be a composite of the namespace's name and the name of the Application, and this can easily exceed the 63 characters length limit imposed on label values. Annotations have a notably greater length limit.

https://argo-cd.readthedocs.io/en/stable/operator-manual/app-any-namespace/#switch-resource-tracking-method

変更する必要はないけど推奨とのこと。
理由は、label の文字数制限が 63 文字ということを挙げています。

複数の Namespace で Application リソースを扱えるように設定すると、デフォルト以外のネームスペースでデプロイすると、tracking 用の名前を `<Namespace>_<アプリケーション名>`といった形式に置き換えるため、文字数制限を超えてしまいデプロイできない可能性が生まれるためです。
そのため、文字数制限が label よりも長い annotations に変更することを推奨していました。

これ以外に別の問題にも当たったので紹介します。

## tracking 用のラベルが汎用的なラベルを使っており、それを変更するｺﾄによる問題

argocd はデフォルトで `app.kubernetes.io/instance`という各所で使われているラベルを追跡用ラベルとして利用しています。

デフォルトではないネームスペースのアプリケーションの場合 `<Namespace>_<アプリケーション名>`という形式に置き換えるという話をしました。

これは文字通り `app.kubernetes.io/instance`の値を置き換えます。
しかし、例えばラベルセレクターで指定している値は変更されません。
そうすると、ラベルセレクターで選択されなくなるのでアプリケーションが正常に動作しなくなります。

## 例: kube-prometheus-stack

kube-prometheus-stack という Helm Chart があります。
これを使うと、Prometheus Operator や NodeExporter など諸々をセットアップすることができます。

Prometheus Operator には ServiceMonitor というリソースがあります。
このリソースは任意の Service リソースを Prometheus で scrape する設定を行うためのリソースです。
この任意の Service リソースを選択するのにラベルセレクターを用いています。

```yaml
  selector:
    matchLabels:
    {{- with .Values.prometheus.monitor.selectorOverride }}
      {{- toYaml . | nindent 6 }}
    {{- else }}
      {{- include "prometheus-node-exporter.selectorLabels" . | nindent 6 }}
    {{- end }}
```

https://github.com/prometheus-community/helm-charts/blob/91325827e969a04be5ff575b95a6d93ccd231095/charts/prometheus-node-exporter/templates/servicemonitor.yaml#L23-L29

デフォルトでは `include "prometheus-node-exporter.selectorLabels"`が展開されます。

この変数は `_helpers.tpl`で定義されています。

```yaml
{{/*
Selector labels
*/}}
{{- define "prometheus-node-exporter.selectorLabels" -}}
app.kubernetes.io/name: {{ include "prometheus-node-exporter.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

https://github.com/prometheus-community/helm-charts/blob/91325827e969a04be5ff575b95a6d93ccd231095/charts/prometheus-node-exporter/templates/_helpers.tpl#L54-L60

このテンプレートを見ると release name が設定されることがわかります。

こんな感じの Application だと `kube-prometheus-stack` (デフォルトで Application の名前) になります。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocds-other-ns
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
  project: default
  sources:
    - chart: kube-prometheus-stack
      repoURL: https://prometheus-community.github.io/helm-charts
      targetRevision: 62.3.1
```

つまり以下のように生成されます。(`app.kubernetes.io/name`はデフォルトでチャート名が入ります)

```yaml
selector:
  matchLabels:
    app.kubernetes.io/name: kube-prometheus-stack
    app.kubernetes.io/instance: kube-prometheus-stack
```

次に ServiceMonitor がマッチさせたい Service の定義を見ます。

```yaml
  labels:
    {{- include "prometheus-node-exporter.labels" $ | nindent 4 }}
    {{- with .Values.service.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
```

https://github.com/prometheus-community/helm-charts/blob/91325827e969a04be5ff575b95a6d93ccd231095/charts/prometheus-node-exporter/templates/service.yaml#L7-L11

`prometheus-node-exporter.labels`変数を見ます。これも `_helpers.tpl`に定義されています。

```yaml
{{/*
Common labels
*/}}
{{- define "prometheus-node-exporter.labels" -}}
helm.sh/chart: {{ include "prometheus-node-exporter.chart" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/component: metrics
app.kubernetes.io/part-of: {{ include "prometheus-node-exporter.name" . }}
{{ include "prometheus-node-exporter.selectorLabels" . }}
{{- with .Chart.AppVersion }}
app.kubernetes.io/version: {{ . | quote }}
{{- end }}
{{- with .Values.commonLabels }}
{{ tpl (toYaml .) $ }}
{{- end }}
{{- if .Values.releaseLabel }}
release: {{ .Release.Name }}
{{- end }}
{{- end }}
```

https://github.com/prometheus-community/helm-charts/blob/91325827e969a04be5ff575b95a6d93ccd231095/charts/prometheus-node-exporter/templates/_helpers.tpl#L34-L52

いろいろありますが、`{{ include "prometheus-node-exporter.selectorLabels" . }}`とあるので以下のように生成されることがわかります。

```yaml
labels:
  ...
  app.kubernetes.io/name: kube-prometheus-stack
  app.kubernetes.io/instance: kube-prometheus-stack
  ...
```

ここで、生成したマニフェストの`metadata.labels."app.kubernetes.io/instance"`を argocd が書き換えるので実際には以下のようなものがデプロイされます。

```yaml
labels:
  ...
  app.kubernetes.io/name: kube-prometheus-stack
  app.kubernetes.io/instance: argocd-other-ns_kube-prometheus-stack
  ...
```

Prometheus は ServiceMonitor を元に`app.kubernetes.io/instance: kube-prometheus-stack`なものを scrape するので、argocd が label を書き換えた Service を scrape してくれないことがわかります。

## まとめ

- ラベルセレクターで使われる可能性が高い `app.kubernetes.io/instance`を tracking label として使用していると、argocd によって書き換えられてラベルセレクターが意図した選択をしなくなります
- これを回避するため、argocd 以外のネームスペースで Application を扱うときは、tracking label を別のものに変更しておくとよさそうです
