# Deploy the Rook Operator



* #### Deploy the Rook Operator

  

* rook-ceph 安装方法按照官方文档进行安装，当前最新release版本是0.9，https://rook.github.io/docs/rook/v0.9/  ，代码地址 https://github.com/rook/rook/tree/release-0.9

* 通过Rancher平台部署的Kubernetes集群部署rook ceph成功后，kubernetes操作rook ceph rbd块存储需要修改rook的配置，否则会造成pod无法连接pvc，出现超时报错，因为找不到相应的驱动。

  * 配置修改说明见官方说明 https://rook.github.io/docs/rook/v0.9/flexvolume.html

  * 需要修改文件operator.yaml开启以下参数并重新发布 

    ```yaml
    env:
    [...]
    - name: FLEXVOLUME_DIR_PATH
      value: "/var/lib/kubelet/volumeplugins"
    ```

