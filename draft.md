# はじめに
この記事では、「**CA 1Day Youth Boot Camp バックエンド/インフラエンジニア編：現場で使うコンテナ技術、Kubernetes＆コンテナ入門**(2021/11/24開催)」に参加したことで得られた知見をまとめます。この記事の対象者としては、

* **kubernetesについてサクッと知りたい**
* **Dockerは触ったことあるけど、k8sは無い**

といった方を想定しています。
私自身も、Dockerは触ったことがある(1ヶ月程度)がk8sは初心者(2日)の状態で上のインターンシップに参加したのですが、非常に勉強になりました。kubernetesに興味がある方の参考になれば幸いです。

# Kubernetes(以下k8s)入門

## 定義
* k8sとは、宣言的な構成管理と自動化を促進し、コンテナ化されたワークロードやサービスを管理するための、ポータブルで拡張性のあるオープンソースのプラットフォーム（[公式ドキュメント](https://kubernetes.io/ja/docs/concepts/overview/what-is-kubernetes/)より）
* 要するに、**コードベースで複数のDockerコンテナを適切に管理してくれるプラットフォーム。**

## Dockerとの関係
* Docker
    * それぞれのアプリケーションごとに**コンテナを作成**
* k8s
    * 複数のホストマシン間でデプロイされた複数の**コンテナの管理**

## メリット
* **大量のコンテナの管理**
    * コードで管理(Infrastructure as Code)するので設定が柔軟に行える。
* **デプロイの高速化**
    * コンテナを元にデプロイを自動で行ってくれるので、手動でデプロイする必要がない。
* **高可用性**
    * k8sはPodという最小単位で構成されているため、Podを増減させることで、可用性を高めることができる。 

他にも様々なメリットがある。

## k8sの概念
k8sを機能させるには、**リソース**（Kubernetes API オブジェクト）を使用して、実行したいアプリケーションやその他のワークロード、使用するコンテナイメージ、レプリカ(複製)の数、どんなネットワークやディスクリソースを利用可能にするかなど、クラスターの **desired state(望ましい状態)**を**yaml**で記述する。desired stateをセットするには、Kubernetes APIを使用してリソースを作成する。通常はコマンドラインインターフェイス `kubectl`を用いて**Kubernetes API**を操作する。

* 概略図（一例）
![概略図](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/888576/bbf4a3dc-34d8-1ef6-11da-c75ec350e076.png)


* **クラスタ**
    * k8sの様々なリソースを管理する集合体（上図の一番外側)
* **コンポーネント**：実行されるプロセス
    * マスターコンポーネント：マスターノードで実行されるもの。クラスターに関する全体的な決定(スケジューリングなど)を行います
        * kube-apiserverver
        * etcd
        * kube-scheduler
        * kube-controller-manage
    * ワーカーコンポーネント：すべてのノードで実行され、稼働中のPodの管理やKubernetesの実行環境を提供する。
        * kube-proxy
        * kubelet
        * コンテナランタイム
* **リソース**
    * クラスタ内の構成要素（上図のNode, Pod）

|リソース名|役割|
|-|-|
|Node|クラスタで実行するコンテナを配置するためのサーバ|
|Namaspace|クラスタ内の仮想クラスタ|
|Pod|コンテナ実行に関する最小単位リソース|
|RepricaSet|同じ仕様のPodを複数生成・管理する|
|Deployment|ReplicaSetの世代管理をする|
|Service|Podの集合にアクセスするための経路を定義|
|Ingress|Serviceをk8sクラスタの外部に公開する|
|...（他にも多くのリソースがある）|...|

k8sでは、これらのリソースがクラスタ内で強調しながらコンテナシステムを構成している。以下では、各リソースにつてい詳しく見ていく。

## 準備
ここからは、実際に手を動かして確認していく。まずは、下準備。

### 準備1

* Dockerをインストールしていない方
    * [こちら](https://www.docker.com/get-started)からインストールする。

* Dockerをインストールしている方
    * Docker for Desktopの設定で、k8sを有効にする
    * [公式ドキュメント](https://kubernetes.io/ja/docs/tasks/tools/install-kubectl/)に従って`kubectl`をインストールする。これを使用することで、k8sクラスターに対してコマンドを実行することができる。
![Screen Shot 2021-11-26 at 1.58.01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/888576/b8fd1026-d555-bbc3-7740-915f985b9e57.png)

```
$ docker version
```
と

```
$ kubectl version
```
を実行し、それぞれバージョン情報が表示されればOK。


### 準備2
サンプルファイルが入っているディレクトリのダウンロード

```
$ git clone https://github.com/CyberAgentHack/one-day-youth-bootcamp-ciu.git

$ $cd Myk8s
```
を実行。
`Myk8s`ディレクトリに入った状態で、進めていく。

### 準備3（任意）
* [こちら](https://github.com/kubernetes/dashboard)に従って、Kubernetes Dashboardのインストール
* さらに、[こちら](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md#creating-sample-user)にあるように`dashboard-adminuser.yaml`を`one-day-youth-bootcamp-ciu`ディレクトリに作成し、`kubectl apply -f dashboard-adminuser.yaml`を実行。
* これで、Kubernetes Dashboardの使用準備が整った。

## Node
Nodeとはクラスタが持つリソースで最も大きい概念。

k8sのクラスタの管理下に登録されているDockerホストのこと。k8sでコンテナをデプロイするために利用される。クラスタはマスターノードとそれ以外で構成される。

Node一覧取得

```
$ kubectl get nodes
NAME             STATUS   ROLES                  AGE    VERSION
docker-desktop   Ready    control-plane,master   3d3h   v1.21.5
```
ローカル環境のk8sであれば、クラスタ作成時（k8sを有効にした時）に作られたVMがNodeの1つとして登録されているらしい。

## Namespace
Namespaceとは、k8sクラスタの内部の仮想的なクラスタ。

Namsespace一覧取得

```
$ kubectl get namespace
NAME                   STATUS   AGE
default                Active   3d23h
kube-node-lease        Active   3d23h
kube-public            Active   3d23h
kube-system            Active   3d23h
kubernetes-dashboard   Active   3d20h
```
`default`: As its name implies, this is the namespace that is referenced by default for every Kubernetes command, and where every Kubernetes resource is located by default. Until new namespaces are created, the entire cluster resides in ‘default’.

* 参考文献
    * [公式](https://kubernetes.io/ja/docs/concepts/overview/working-with-objects/namespaces/#namespace%E3%82%92%E5%88%A9%E7%94%A8%E3%81%99%E3%82%8B)
    * [参考資料](https://speakerdeck.com/bo0km4n/ca-1day-youth-bootcamp-ciu-kubernetes?slide=19)

## Pod
Podとは、コンテナの集合体。少なくとも１つのコンテナを持つ。

Pod一覧取得

```
$ kubectl get pod
No resources found in default namespace.![pod.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/888576/e2e65ee8-237d-7dfc-ea20-b252a604153a.png)
```
Podを作成していないので、default namespaceにはPodが存在しない。



試しに、[こちら]()のマニュフェストファイル（リソースを定義したファイル）をもとに、Podを作成する。

ファイルの説明（詳しくは、[こちら](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#pod-v1-core)）

```yml
apiVersion: v1 # どのバージョンのKubernetesAPIを利用してリソースを作成するか
kind: Pod # リソースの種類
metadata: # リソースを一意に特定するための情報
  name: nginx # 文字列を指定
spec: # リソースの望ましい状態（specの正確なフォーマットは、リソースごとに異なる。cf. 参考文献 マニュフェストファイルの説明）
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

```
$ kubectl apply -f pod.yml

$ kubectl get pod
NAME       READY   STATUS    RESTARTS   AGE
echo-pod   1/1     Running   0          20s
```
無事に作成できている。
k8s Dashboardで、default NamespaceのPodsを確認してみると、
![Screen Shot 2021-11-29 at 11.51.07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/888576/13a3ce6f-15fd-49c8-5b24-30a20d8179fe.png)
となっている。
整理すると、

```
$ kubectl config current-context 
docker-desktop 
```
より、接続しているクラスタは`docker-desktop`であるとわかる。そして、そのクラスタ内のNode一覧は、

```
$ kubectl get nodes
NAME             STATUS   ROLES                  AGE    VERSION
docker-desktop   Ready    control-plane,master   3d3h   v1.21.5
```
より、マスターノードだけ。
さらに、Namespace一覧は、

```
$ kubectl get namespace
NAME                   STATUS   AGE
default                Active   4d3h
kube-node-lease        Active   4d3h
kube-public            Active   4d3h
kube-system            Active   4d3h
kubernetes-dashboard   Active   4d
```
であるとわかる。
今回、Podをapplyすると、１つしかないNode（マスターノード）にPodが作成され、k8sのアーキテクチャー概略図は、次のようになっていると考えられる。
![pod.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/888576/68cf4d03-60dd-2969-7ae4-52131c08931d.png)

ただし、Namespaceは省略した。

* 参考文献
    * [公式](https://kubernetes.io/docs/concepts/workloads/pods/)

## RepricaSet
RepricaSetの目的は、どのような時でも安定したレプリカPodのセットを維持すること。指定された理想のレプリカ数にするためにPodの作成と削除を行うことにより、その目的を達成する。新しいPodを作成するとき、Podテンプレートを使用する。

RepricaSet一覧取得

```
$ kubectl get rs
No resources found in default namespace.
```
RepricaSetを作成していないので、default namespaceにはRepricaSetが存在しない。

試しに、[こちら]()のマニュフェストファイルをもとに、RepricaSetを作成する。

```
$ kubectl apply -f rs.yml

$ kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       12s

$ kubectl get pod
NAME             READY   STATUS    RESTARTS   AGE
frontend-b9dqp   1/1     Running   0          27s
frontend-h75sb   1/1     Running   0          27s
frontend-lppc7   1/1     Running   0          27s
```
無事に作成できている。
ファイルの説明（詳しくは、[こちら](https://kubernetes.io/ja/docs/concepts/workloads/controllers/replicaset/#replicaset%E3%81%AE%E3%83%9E%E3%83%8B%E3%83%95%E3%82%A7%E3%82%B9%E3%83%88%E3%82%92%E8%A8%98%E8%BF%B0%E3%81%99%E3%82%8B))

```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata: # リソースを一意に特定するための情報
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec: # ReplicaSetの望ましい状態を定義
  replicas: 3 # Podの数
  selector: # .spec.template.metadata.labelsと一致させる。ReplicaSetが所有するPodを指定するため。（今回の場合、tier: frontendが一致している）
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template: # template以下はPodの定義と同様に書く。templateの定義を持つPodの複製を行う。
    metadata:
      labels: # spec.selectorと一致させる（2つのラベルのうちtier: frontendが一致している）
        app: guestbook
        tier: frontend
    spec: # Podの望ましい状態を定義
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access environment variables to find service host
          # info, comment out the 'value: dns' line above, and uncomment the
          # line below.
          # value: env
        ports:
        - containerPort: 80
```

* 参考文献
    * [公式](https://kubernetes.io/ja/docs/concepts/workloads/controllers/replicaset/)
    * [youtube](https://youtu.be/r0NpHb-6IvY?t=1220)
    
## Deployment
Deploymentは、ReplicaSetを管理・操作するためのリソース。実用上では、ReplicaSetを直接用いるのではなく、Deploymentのマニュフェストファイルを扱うことが多いらしい。

Deployment一覧取得

```
$ kubectl get deployments
No resources found in default namespace.
```
Deploymentを作成していないので、default namespaceにはDeploymentが存在しない。

試しに、[こちら]()のマニュフェストファイルをもとに、Deploymentを作成する。

ファイルの説明（詳しくは、[こちら](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#deployment-v1-apps)）

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec: # Deploymentの望ましい状態を定義
  replicas: 3 # 3つのレプリカPodを作成
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

作成コマンド

```
$ kubectl apply -f dep.yaml

$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           4m48s

$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-66b6c48dd5   3         3         3       5m4s

$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-66b6c48dd5-6jt7m   1/1     Running   0          5m46s
nginx-deployment-66b6c48dd5-89frd   1/1     Running   0          5m46s
nginx-deployment-66b6c48dd5-m98mj   1/1     Running   0          5m46s
```
となり、宣言通りのリソースが作成されている。

* 参考文献
    * [公式](https://kubernetes.io/ja/docs/concepts/workloads/controllers/deployment/)

## Service
Serviceとは、k8sクラスタ内で、Podの集合（主にReplicaSet）に対する経路やサービスディスカバリを提供するためのリソース。要は、**k8sクラスタ内のネットワークをいい感じに捌いてくれるリソース。**

Serviceには様々な種類がある。

* **ClusterIP**(デフォルト)
    * クラスター内部のIPでServiceを公開する。Serviceはクラスター内部からのみ疎通性があります。
* **NodePort**
    * 各NodeのIPにて、静的なポート(NodePort)上でServiceを公開する。そのNodePort のServiceが転送する先のClusterIP Serviceが自動的に作成される。`<NodeIP>:<NodePort>`にアクセスすることによってNodePort Serviceにアクセスできるようになる。
* **LoadBalancer**
    * クラウドプロバイダーのロードバランサーを使用して、Serviceを外部に公開する。クラスター外部にあるロードバランサーが転送する先のNodePortとClusterIP Serviceは自動的に作成される。
* **ExternalName**
    * CNAMEレコードを返すことにより、externalNameフィールドに指定したコンテンツ(例: foo.bar.example.com)とServiceを紐づける。しかし、いかなる種類のプロキシーも設定されない。

### ClusterIP
まず、ClusterIPについて見ていく。

Serviceのマニュフェストファイル

```yml
apiVersion: v1
kind: Service
metadata:
  name: echoserver
spec:
  ports:  # The list of ports that are exposed by this service.
  - port: 80 # The port that will be exposed by this service.
    targetPort: 8080 # ターゲットとなるPodのポート番号
    protocol: TCP # 
  selector:
    app: echoserver #　このラベルと一致するPodがserviceのターゲットとなり、serviceを経由してtcpリクエストが流れる 
```
まず、

```
$  kubectl apply -f dep2.yml
```
で、予めPodを作っておく。

```
$ kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
echoserver-8499b7dffc-24w99   1/1     Running   0          37m
echoserver-8499b7dffc-7fn2h   1/1     Running   0          37m
echoserver-8499b7dffc-9mfsr   1/1     Running   0          37m
```

```
# Service をデプロイ
$ kubectl apply -f svc.yml

# 割り振られた IP を確認。
$ kubectl describe svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
echoserver   ClusterIP   10.98.108.161   <none>        80/TCP    37m
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   6d23h
```
ここで、ClusterIPは、クラスタ内部で䛾み使用可能な仮想IPである。 
サービスにアクセスするために、 クラスタ内に適当にPodを建ててsrviceの80番ポートにアクセスする（マニュフェストファイルで、.spec.ports.port=80としたため）

```
$ kubectl run --image=centos:6 --restart=Never --rm -i testpo -- curl -s http://[svc-ip]:80
foo
pod "testpo" deleted
```
と表示されればOK。（ただし、[svc-ip] = 10.98.108.16, [fooと表示される理由]()）
ここで、ClusterIPはデプロイのたびに変わるので、その度にいちいち調べるのは面倒。そこで、[こちら](https://kubernetes.io/ja/docs/concepts/services-networking/dns-pod-service/#services)にあるように、DNS名を用いてアクセスしてみる。

```
$ kubectl run --image=centos:6 --restart=Never --rm -i testpo -- curl -s http://echoserver.default.svc.cluster.local:80
foo
pod "testpo" deleted
```
と表示されればOK。

整理すると、クラスタ内に適当に建てたPodから、`http://[svc-ip]:80`または`http://echoserver.default.svc.cluster.local:80`でServiceにアクセスすると、マニュフェストファイル通り、`app: echoserver`のラベルを持つPodの`8080`ポートにリクエストが流れ、`foo`が表示される。

### NodePort
次に、NodePortについて見ていく。

Serviceのマニュフェストファイル

```yml
apiVersion: v1
kind: Service
metadata:
  name: echoserver
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: NodePort # type determines how the Service is exposed. Defaults to ClusterIP. Valid options are ExternalName, ClusterIP, NodePort, and LoadBalancer.
  selector:
    app: echoserver
```

先ほどと同様に、予めPodを作っておく。（作っていない場合）

```
# Service をデプロイ
$ kubectl apply -f svc_node_port.yml

# 割り振られた IP を確認。
$ kubectl describe svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
echoserver   NodePort    10.105.66.102   <none>        80:32694/TCP   32s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        7d
```
NodePortサービスを作成した場合、`80:32694/TCP`とあるように、ノードの`32694`ポートからServiceへアクセスできる。これにより、Serviceをk8sクラスタの外に公開できる。

実際、ローカル（クラスタ外）からService（クラスタ内）にアクセスできる。

```
$ curl http://localhost:32694
foo
```
また、

適当に建てたPod(クラスタ内)からもService（クラスタ内）にアクセスできる。

```
$ kubectl run busybox -it --rm --image=busybox --restart=Never -- /bin/sh -c "wget -q -O- [NodeIP]:30937"
foo
```

([NodeIP]=`$ kubectl get node -owide`コマンドで表示される`INTERNAL-IP`)
ともできる。

整理すると、クラスタ内外からServiceにアクセスすると、マニュフェストファイル通り、app: echoserverのラベルを持つPodの8080ポートにリクエストが流れ、`foo`が表示される。






















## Ingress
Ingressは、クラスター内のServiceに対する**外部からのアクセス**(主にHTTP)を管理するリソース。NodePortはL4層（トランズポート層）までしか扱えないが、IngressはL7層（アプリケーション層）まで制御できる。

[公式](https://kubernetes.io/ja/docs/concepts/services-networking/ingress/#ingress%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B%E4%B8%8A%E3%81%A7%E3%81%AE%E5%89%8D%E6%8F%90%E6%9D%A1%E4%BB%B6)にもあるように、ローカルk8s環境ではIngressを使用してServiceを公開することができない。
[こちら](https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md)に従って、nginx_ingress_controllerをデプロイする。

しばらくすると、

```
$ kubectl get pods --namespace=ingress-nginx
NAME                                      READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-54bfb9bb-qtcsb   1/1     Running   0          5m19s
```
のように、デプロイされている。

次に、























# 参考文献
* [[挫折したエンジニア向け] Kubernetesの仕組みをちゃんと理解する](https://youtu.be/r0NpHb-6IvY)⇦ざっくり理解できる
* リソースごとの公式ドキュメント
    * [Pod](https://kubernetes.io/ja/docs/concepts/workloads/pods/pod-overview/)
    * [ReplicaSet](https://kubernetes.io/ja/docs/concepts/workloads/controllers/replicaset/)
    * [Service](https://kubernetes.io/ja/docs/concepts/services-networking/service/)
    * [Deployment](https://kubernetes.io/ja/docs/concepts/workloads/controllers/deployment/)
    * [Ingress](https://kubernetes.io/ja/docs/concepts/services-networking/ingress/)
* マニュフェストファイルの詳細
    * [ラベル(Labels)とセレクター(Selectors)](https://kubernetes.io/ja/docs/concepts/overview/working-with-objects/labels/)
    * [Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#pod-v1-core)
    * [ReplicaSet](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#replicaset-v1-apps)
    * [Deployment](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#deployment-v1-apps)
    * [Service](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#service-v1-core)
    * [Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#ingress-v1-networking-k8s-io)
* [インターンで用いた資料](https://speakerdeck.com/bo0km4n/ca-1day-youth-bootcamp-ciu-kubernetes?slide=19)
* [Docker/Kubernetes実践コンテナ開発入門
](https://www.amazon.co.jp/Docker-Kubernetes-%E5%AE%9F%E8%B7%B5%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E9%96%8B%E7%99%BA%E5%85%A5%E9%96%80-%E5%B1%B1%E7%94%B0-%E6%98%8E%E6%86%B2/dp/4297100339)（書籍）




