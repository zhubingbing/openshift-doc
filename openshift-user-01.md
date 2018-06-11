## openshift实践-权限资源管理

更多更详细的介绍--官网文档：https://docs.openshift.org/latest/admin_guide/index.html


这里不涉及到认证登录的介绍，openshfit支持很多认证方式，比如AllowAll，CA认证, HTPasswd, KeyStone, LDAP, Oauth等
这里为了简化，用默认的AllowAll来做权限控制的测试和说明。

权限管理，即访问API资源之前，必须要经过的访问策略校验，主要分为5种： AlwaysDeny、AlwaysAllow（默认）、ABAC、RBAC、Webhook

主要说明user, group, rule，role，policy，policybinding之间的关系，以及提出这些概念，各自是为了解决什么问题。

1. user:  说到user其实就是一个用户账号(userAccount)，用它来和k8s集群和openshift集群做交互（登录，kubectl, oc等）， 但还有一个容易混淆的概念就是sercieAccount，有了userAccount为什么还又来个serviceAccount的设计， 这两者有什么区别 ？ 以下是kubernetes官方对两者的解释

	user account是为人类设计的，而service account则是为跑在pod里的进程用的，运行在pod里的进程需要调用Kubernetes API以及非Kubernetes API的其它服务（如image repository/被mount到pod上的NFS volumes中的file等）;

	user account是global的，即跨namespace使用；而service account是namespaced内的，即仅在所属的namespace下使用;

	user account可能会涉及到很多权限设定和商业逻辑在里面，而后者是更轻量级的，是集群用户针对某namespace内的服务使用的，一般遵循最小特权原则，如监控服务需要访问APIsever等;

	useraccount需要借助第三方实现，后者系统都会默认在namesspace里创建default，亦可自定义.
   总结下来： userAccount 只为用户（人）设计的， 跨namespaces，具有很多权限设定。
            sercieAccount 是为了访问集群服务设计的（例如: pod里面的服务需要访问集群apiserver). 在namespaces里面使用,遵循最小特权原则。

    两者大部分流程是一致的，都是要先认证通过再校验权限，然后才是action， 实际上一般是由userAccount来控制serviceAccount来完成特定的任务， 比如一个用户A自建了服务1和服务2， 但只想把服务2开发给用户B，这样的serviceAccount就可以排上用场了, 又或者我有几个服务，有了serviceAccount就可以来限制用户的访问权限（list, watch, update, delete）了
     
2. group: 针对user的权限批量操作而设计。例如1个组(group)可以有多个user， 1个user可以有多个组. 每个组代表一组特定的用户

3. rule: 是规则, 是对一组对象上被允许的动作（get, list, create, update, delete, deletecollection 和 watch）描述, 可操作对象主要是 container，images，pod，servcie， project， user， build， imagestream， dc， route， templeate.

4. role: 就是规则的集合，俗称角色， 不同对象上的不同动作，可以任意组成各种角色，系统默认的有 admin basic-user cluster-admin cluster-admin edit self-provisioner view

```
## kubectl和oc命令都可以， 查看集群roles
[root@localhost net.d]# oc get  roles --all-namespaces
NAMESPACE                           NAME
kube-public                         system:controller:bootstrap-signer
kube-service-catalog                configmap-accessor
kube-system                         extension-apiserver-authentication-reader
kube-system                         system::leader-locking-kube-controller-manager
kube-system                         system::leader-locking-kube-scheduler
kube-system                         system:controller:bootstrap-signer
kube-system                         system:controller:cloud-provider
kube-system                         system:controller:token-cleaner
openshift-node                      system:node-config-reader

## 查看roles的定义
[root@localhost net.d]# kubectl describe role system:controller:cloud-provider -n kube-system
Name:         system:controller:cloud-provider
Namespace:    kube-system
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  openshift.io/reconcile-protect=false
API Version:  authorization.openshift.io/v1
Kind:         Role
Metadata:
  Creation Timestamp:  2018-06-06T07:01:04Z
  Resource Version:    273
  Self Link:           /apis/authorization.openshift.io/v1/namespaces/kube-system/roles/system%3Acontroller%3Acloud-provider
  UID:                 60d5c7a0-6957-11e8-9a13-0800272eeb78
Rules:
  API Groups:

  Attribute Restrictions:  <nil>
  Resources:
    configmaps
  Verbs:
    create
    get
    list
    watch
Events:  <none>
```

5. policy: 策略, 保存特定namespace的所有角色roles的对象。 每个命名空间最多只有一个Policy策略

6. rolebinding: 就是把user或者group与角色role进行关联，注意. user和group可以被关联到多个roles

7. pollicybing, 就是就是多个rolebindings的描述

policy的提出主要是为了区分cluster-policy和local-policy的。

cluster policy是适用于所有namespace的角色和绑定； local policy则是试用于具体的某个namespace的；

以上可以通过oc describe clusterPolicy default来看查看所有详细的信息；

还可以通过oc policy who-can <动作> <资源对象>， 比如说查看谁能get pod之类的，就是oc policy who-can get pod
```
[root@localhost net.d]# oc policy who-can get pod
Namespace: hello
Verb:      get
Resource:  pods

Users:  system:admin
        system:kube-scheduler
        system:serviceaccount:default:router
        system:serviceaccount:hello:deployer
        system:serviceaccount:kube-service-catalog:default
        system:serviceaccount:kube-system:clusterrole-aggregation-controller
        system:serviceaccount:kube-system:deployment-controller
        system:serviceaccount:kube-system:endpoint-controller
        system:serviceaccount:kube-system:generic-garbage-collector
        system:serviceaccount:kube-system:namespace-controller
        system:serviceaccount:kube-system:persistent-volume-binder
        system:serviceaccount:kube-system:statefulset-controller
        system:serviceaccount:management-infra:management-admin
        system:serviceaccount:openshift-ansible-service-broker:asb
        system:serviceaccount:openshift-infra:build-controller
        system:serviceaccount:openshift-infra:deployer-controller
        system:serviceaccount:openshift-infra:pv-recycler-controller
        system:serviceaccount:openshift-infra:sdn-controller
        system:serviceaccount:openshift-infra:template-instance-controller

Groups: system:cluster-admins
        system:cluster-readers
        system:masters
```

如果openshift自带的角色不能满足的话，还可以自定义角色role
```
$ oc get clusterrole view -o yaml > clusterrole_view.yaml
$ cp clusterrole_view.yaml localrole_exampleview.yaml
$ vim localrole_exampleview.yaml
# 1. Update kind: ClusterRole to kind: Role
# 2. Update name: view to name: exampleview
# 3. Remove resourceVersion, selfLink, uid, and creationTimestamp
$ oc create -f path/to/localrole_exampleview.yaml -n <project_you_want_to_add_the_local_role_exampleview_to>
```


### 用户管理

1.  使用集群管理员登陆集群

```
oc login -u system:admin
```

2. 用户相关操作
```
## 创建用户
oc create user zhubingbing

## 创建一个组

oc adm groups new test

## 查看组
oc get groups

## 把集群role添加到user
oc adm policy add-cluster-role-to-user system:admin zhubingbing

## 把user添加到groups
oc adm groups add-users test zhubingbing

```
