# Deploy Rook



* #### k8s中正确删除一个pod

  ```shell
  删除pod对应的deployment,
  否则只是删除pod是不管用的，还会看到pod，因为deployment.yaml文件中定义了副本数量.
  
  ```

  [root@test2 ~]# kubectl get pod -n jenkins
  NAME                        READY     STATUS    RESTARTS   AGE
  jenkins2-8698b5449c-dbqqb   1/1       Running   0          8s
  [root@test2 ~]# 

  删除deployment

  [root@test2 ~]# kubectl get deployment -n jenkins
  NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  jenkins2   1         1         1            1           17h
  [root@test2 ~]# kubectl delete deployment jenkins2 -n jenkins

  再次查看pod消失

  deployment.extensions "jenkins2" deleted
  [root@test2 ~]# kubectl get deployment -n jenkins
  No resources found.
  [root@test2 ~]# 
  [root@test2 ~]# kubectl get pod -n jenkins
  No resources found.



* #### Deploy the Rook Operator

  

* ###### Alter operator.yaml file

* rook-ceph 安装方法按照官方文档进行安装，当前最新release版本是0.9，https://rook.github.io/docs/rook/v0.9/  ，代码地址 https://github.com/rook/rook/tree/release-0.9

* 在Rancher平台安装成功后，kubernetes操作rook ceph rbd块存储需要修改配置，否则会造成pod无法连接pvc，出现超时报错。

  * 配置修改说明见地址 https://rook.github.io/docs/rook/v0.9/flexvolume.html

  * 需要修改文件operator.yaml开启以下参数并重新发布 

    ```yaml
    env:
    [...]
    - name: FLEXVOLUME_DIR_PATH
      value: "/var/lib/kubelet/volumeplugins"
    ```

