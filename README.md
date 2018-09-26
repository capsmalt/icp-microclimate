# icp-microclimate

########## microclimate install #############
# Microclimate

## Microclimate setup
- Microclimate自体をデプロイするNamespaceを作成
```
# kubectl create namespace microclimate
namespace/microclimate created
```

- Namespaceの一覧に "microclimate" が作成されていることを確認
```
# kubectl get namespaces
NAME           STATUS    AGE
cert-manager   Active    15h
default        Active    15h
ibmcom         Active    15h
istio-system   Active    15h
kube-public    Active    15h
kube-system    Active    15h
microclimate   Active    11s
platform       Active    14h
services       Active    14h
```

> Hint:
kubectlコマンド実行時のNamespaceを事前にcontextに設定しておくことで，各種操作時にNamespaceの指定を省略できます。以下の場合は，"--namespace microclimate" を省略できます。
kubectlが使用するnamespaceを設定する
```
kubectl config set-context $(kubectl config current-context) --namespace microclimate
```

- Microclimate pipelineがデプロイするアプリケーション用のNamespaceを作成
```
# kubectl create namespace microclimate-pipeline-deployments
namespace/microclimate-pipeline-deployments created
```

- クラスターのイメージ取得のポリシー(clusterimagepolicy)に追加リポジトリを記載
```
# kubectl edit clusterimagepolicy ibmcloud-default-cluster-image-policy
以下を追加
  - name: docker.io/maven:*
  - name: docker.io/lachlanevenson/k8s-helm:*
  - name: docker.io/jenkins/*
```

- 2つのDocker registry用のSecretを作成 (Microclimate用途，Pipelineがアプリデプロイする用途)
 - microclimate-registry-secret (Microclimate用途)
 ```
 # kubectl create secret docker-registry microclimate-registry-secret   --docker-server=mycluster.icp:8500 --docker-username=admin --docker-password=admin --docker-email=null --namespace=microclimate
 secret/microclimate-registry-secret created
 ```
 - microclimate-pipeline-secret (Pipelineがアプリデプロイする用途)
 ```
 # kubectl create secret docker-registry microclimate-pipeline-secret   --docker-server=mycluster.icp:8500 --docker-username=admin --docker-password=admin --docker-email=null --namespace=microclimate-pipeline-deployments
 secret/microclimate-pipeline-secret created
 ```
   - "default" Service Account の "imagePullSecret" が既に連携されているか確認
	```
	# kubectl describe serviceaccount default --namespace microclimate-pipeline-deployments
	Name:                default
	Namespace:           microclimate-pipeline-deployments
	Labels:              <none>
	Annotations:         <none>
	Image pull secrets:  microclimate-pipeline-secret
	Mountable secrets:   default-token-jbqzg
	Tokens:              default-token-jbqzg
	Events:              <none>
	```
	 - 他のSecretを含んでいなければ，以下のコマンドでService Accountにパッチする
	```
	# kubectl patch serviceaccount default --namespace microclimate-pipeline-deployments -p '{"imagePullSecrets": [{"name": "microclimate-pipeline-secret"}]}'
	serviceaccount/default patched
	```

- MicroclimateがHelmをセキュアに使用するためのSecretを作成
```
# kubectl create secret generic microclimate-helm-secret --from-file=cert.pem=$HELM_HOME/cert.pem --from-file=ca.pem=$HELM_HOME/ca.pem --from-file=key.pem=$HELM_HOME/key.pem --namespace microclimate
secret/microclimate-helm-secret created
```

- パーシスタンスストレージの構成
 - ファイルシステム上のフォルダを作成
 ```
 # mkdir -p /microclimate/mc-data
 # mkdir -p /microclimate/mc-jenkins
 # chmod 777 /microclimate/mc-data
 # chmod 777 /microclimate/mc-jenkins
 ```
 - PersistantVolumeのとPersistantVolumeCraimの作成
 	- mc-data
		- PV
			mc-data
			8Gi
			ReadWriteMany
			hostpath
			path: /microclimate/mc-data
		- PVC
			mc-data
			Namespace: microclimate
			8Gi
			ReadWriteMany

	- mc-jenkins
		- PV
			mc-jenkins
			8Gi
			ReadWriteOnce
			hostpath
			path: /microclimate/mc-jenkins
		- PVC
			mc-jenkins
			Namespace: microclimate
			8Gi
			ReadWriteOnce

- ICPクラスターにログイン
```
# cloudctl login -a https://169.62.93.170:8443 --skip-ssl-validation
kube-systemを選択
```

- ibm-chartsのhelmリポジトリを追加
```
# helm repo add ibm-charts https://raw.githubusercontent.com/IBM/charts/master/repo/stable/
"ibm-charts" has been added to your repositories
```

- Microclimateをインストール
```
# helm install --name microclimate --namespace microclimate --set global.rbac.serviceAccountName=micro-sa,jenkins.rbac.serviceAccountName=pipeline-sa,hostName=microclimate.169.62.93.163.nip.io,jenkins.Master.HostName=jenkins.169.62.93.163.nip.io,persistence.existingClaimName=mc-data,jenkins.Persistence.ExistingClaim=mc-jenkins ibm-charts/ibm-microclimate --tls
```

実行例と出力結果
```
# helm install --name microclimate --namespace microclimate --set global.rbac.serviceAccountName=micro-sa,jenkins.rbac.serviceAccountName=pipeline-sa,hostName=microclimate.169.62.93.163.nip.io,jenkins.Master.HostName=jenkins.169.62.93.163.nip.io,persistence.existingClaimName=mc-data,jenkins.Persistence.ExistingClaim=mc-jenkins ibm-charts/ibm-microclimate --tls
NAME:   microclimate
LAST DEPLOYED: Mon Sep 24 15:30:40 2018
NAMESPACE: microclimate
STATUS: DEPLOYED

RESOURCES:
==> v1beta1/ClusterRole
NAME                                     AGE
microclimate-ibm-microclimate-cr-devops  1m
microclimate-ibm-microclimate-cr-micro   1m

==> v1beta1/Ingress
NAME                           HOSTS                              ADDRESS        PORTS    AGE
microclimate-jenkins           jenkins.169.62.93.163.nip.io       169.62.93.163  80, 443  1m
microclimate-ibm-microclimate  microclimate.169.62.93.163.nip.io  169.62.93.163  80, 443  1m

==> v1beta1/ClusterRoleBinding
NAME                                      AGE
microclimate-ibm-microclimate-crb-devops  1m
microclimate-ibm-microclimate-crb-micro   1m

==> v1/Service
NAME                                  TYPE       CLUSTER-IP  EXTERNAL-IP  PORT(S)                     AGE
microclimate-jenkins-agent            ClusterIP  10.0.0.95   <none>       50000/TCP                   1m
microclimate-jenkins                  ClusterIP  10.0.0.175  <none>       8080/TCP                    1m
microclimate-ibm-microclimate-devops  ClusterIP  10.0.0.129  <none>       9191/TCP                    1m
microclimate-ibm-microclimate         ClusterIP  10.0.0.54   <none>       4191/TCP,9090/TCP,9091/TCP  1m

==> v1beta1/Deployment
NAME                                  DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
microclimate-jenkins                  1        1        1           0          1m
microclimate-ibm-microclimate-devops  1        1        1           0          1m
microclimate-ibm-microclimate         1        1        1           1          1m

==> v1/Pod(related)
NAME                                                  READY  STATUS             RESTARTS  AGE
microclimate-jenkins-68957648b6-vgmsc                 0/1    Running            0         1m
microclimate-ibm-microclimate-devops-584bccd54-mbf52  0/1    ContainerCreating  0         1m
microclimate-ibm-microclimate-66848cc6f8-ptnxc        1/1    Running            0         1m

==> v1/Secret
NAME                           TYPE    DATA  AGE
microclimate-jenkins           Opaque  2     1m
microclimate-ibm-microclimate  Opaque  3     1m

==> v1/ConfigMap
NAME                                                 DATA  AGE
microclimate-jenkins                                 7     1m
microclimate-jenkins-tests                           1     1m
microclimate-ibm-microclimate-create-mc-secret       1     1m
microclimate-ibm-microclimate-create-target-ns       1     1m
microclimate-helmtest-devops                         1     1m
microclimate-ibm-microclimate-jenkinstest            1     1m
microclimate-ibm-microclimate-fixup-jenkins-ingress  1     1m

==> v1/ServiceAccount
NAME         SECRETS  AGE
pipeline-sa  1        1m
micro-sa     1        1m


NOTES:
ibm-microclimate-1.6.0

1. Access the Microclimate portal at the following URL: https://microclimate.169.62.93.163.nip.io

Target namespace set to: microclimate-pipeline-deployments, please verify this exists before creating pipelines
```

 - MicroclimateのIngress hostnameを確認
 ```
 # kubectl get ingress -l release=microclimate -n microclimate
 ```

## Helm setup
- refs: [Setting up the Helm CLI](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/app_center/create_helm_cli.html)

- ICPクラスターにCLIでログインし，証明書などを取得
```
# cloudctl login -a https://<cluster_host_name>:8443 --skip-ssl-validation
	Login ==> admin/admin
	namespace ==> kube-system
```

1行実行例
```
# cloudctl login -a https://169.62.93.163:8443 -u admin -p admin -c id-mycluster-account -n kube-system --skip-ssl-validation
```

```
実行例
# cloudctl login -a https://169.62.93.170:8443 --skip-ssl-validation

Username> admin

Password>
Authenticating...
OK

Select an account:
1. mycluster Account (id-mycluster-account)
Enter a number> 1
Targeted account mycluster Account (id-mycluster-account)

Select a namespace:
1. cert-manager
2. default
3. ibmcom
4. istio-system
5. kube-public
6. kube-system
7. platform
8. services
Enter a number> 6
Targeted namespace kube-system

Configuring kubectl ...
Property "clusters.mycluster" unset.
Property "users.mycluster-user" unset.
Property "contexts.mycluster-context" unset.
Cluster "mycluster" set.
User "mycluster-user" set.
Context "mycluster-context" created.
Switched to context "mycluster-context".
OK

Configuring helm: /root/.helm
OK
root@k8s-icp-00:~/.helm# ls
ca.pem  cache  cert.pem  key.pem  plugins  repository  starters
```

[refs](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.0/manage_cluster/install_cli.html)

- helmの初期構成
```
# export HELM_HOME=~/.helm
# helm init --client-only
```

```
# helm version --tls
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1+icp", GitCommit:"843201eceab24e7102ebb87cb00d82bc973d84a7", GitTreeState:"clean"}
```
