### 🔹 **シナリオ例**

あなたが Kubernetes に次のような Pod を作成しようとしています：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: nginx
```

この時、**Mutating Webhook が「この Pod に annotation を自動的に追加する」** という役割を持っていたとしましょう。

---

### 🔹 **処理のフロー図**

```
あなた (kubectl apply)
   ↓
kube-apiserver に Pod 作成リクエスト送信
   ↓
kube-apiserver:
  認証・認可確認
   ↓
Admission コントローラーが起動
   ↓
MutatingAdmissionWebhook が登録されていれば...
  ↓
外部 Webhook サーバーへ Pod 情報を HTTP POST で送信
  ↓
Webhook サーバー側が Pod を「mutation」した JSON を返信
  （例: annotations を追加した Pod に書き換えて返す）
   ↓
kube-apiserver がその変わった Pod を etcd に保存
```

---

### 🔹 **どんなことが行われるのか（具体例付き）**

#### 1️⃣ **あなたが Pod を作成**

```bash
kubectl apply -f pod.yaml
```

---

#### 2️⃣ **kube-apiserver にリクエストが届く**

- `kube-apiserver` はまず認証・認可を確認。
- その後、Admission Controller の段階で `MutatingAdmissionWebhook` をチェック。

---

#### 3️⃣ **Mutating Webhook サーバーに HTTP で送信**

- kube-apiserver は次のようなリクエストを外部サーバーに送る：

  ```json
  {
    "request": {
      "uid": "12345",
      "kind": { "kind": "Pod" },
      "object": {
        "metadata": {
          "name": "myapp"
        },
        "spec": {
          "containers": [{ "name": "myapp", "image": "nginx" }]
        }
      }
    }
  }
  ```

---

#### 4️⃣ **Webhook サーバーが Pod を「修正」して応答**

- 例えば次のような `patch` を返す：

  ```json
  {
    "response": {
      "uid": "12345",
      "allowed": true,
      "patchType": "JSONPatch",
      "patch": "<base64 encoded patch>"
    }
  }
  ```

  例の patch 内容：

  ```json
  [
    {
      "op": "add",
      "path": "/metadata/annotations/injected-by",
      "value": "my-webhook"
    }
  ]
  ```

  ➡️ つまり「`metadata.annotations` に `injected-by=my-webhook` を追加してください」という指示。

---

#### 5️⃣ **kube-apiserver が修正を適用**

- `kube-apiserver` は patch を適用し、Pod オブジェクトが次のように変わる：

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp
    annotations:
      injected-by: my-webhook
  spec:
    containers:
      - name: myapp
        image: nginx
  ```

---

#### 6️⃣ **最終的に etcd に保存**

- この修正済みの Pod が etcd に保存され、クラスター内で実際にスケジューリング・起動される。

---

### 🔔 **ポイント**

- Webhook サーバー自体は「独立した HTTP サーバー」。

  - 例えば別の Pod としてクラスター内にデプロイされることが多い。

- kube-apiserver は「Webhook サーバーに相談して、必要なら patch を適用してから etcd に保存する」だけ。

---

### 💡 **要約**

✅ まとめると：

- **Mutating Webhook は、API Server に送られた「リソース作成・変更リクエスト」の内容を etcd に保存する前に外部の HTTP サーバーに問い合わせて、内容を修正するための仕組み。**
- kube-apiserver の中に Webhook サーバーが内蔵されているわけではない。
- 具体的な例では「Pod に annotation を自動追加する」「サイドカーコンテナを自動挿入する」などに使われる。

---

Tips：下記も調べてみたい
👉 **Validating Webhook との違い**
👉 **どうやって Webhook が登録されているのか（MutatingWebhookConfiguration など）**

## 補足

EKS Pod Identity Webhook サーバーの場合は、「Pod を作る時に、AWS の認証情報を取得するために Pod の設定変更」をします
