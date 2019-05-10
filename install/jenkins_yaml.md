apiVersion: apps/v1beta2

kind: Deployment

metadata:

  labels:

​    cattle.io/creator: norman

​    workload.user.cattle.io/workloadselector: deployment-cicd-jenkins

  name: jenkins

  namespace: cicd

spec:

  progressDeadlineSeconds: 600

  replicas: 1

  revisionHistoryLimit: 10

  selector:

​    matchLabels:

​      workload.user.cattle.io/workloadselector: deployment-cicd-jenkins

  strategy:

​    rollingUpdate:

​      maxSurge: 0

​      maxUnavailable: 1

​    type: RollingUpdate

  template:

​    metadata:

​      labels:

​        workload.user.cattle.io/workloadselector: deployment-cicd-jenkins

​    spec:

​      containers:

​      \- image: reg.ecs.local/eos/jenkinsci/blueocean-ecs:1.14.0

​        imagePullPolicy: Always

​        name: jenkins

​        ports:

​        \- containerPort: 8080

​          name: 8080tcp01

​          protocol: TCP

​        \- containerPort: 50000

​          name: 50000tcp02

​          protocol: TCP

​        \- containerPort: 8080

​          name: 8080tcp02

​          protocol: TCP

​        resources: {}

​        securityContext:

​          allowPrivilegeEscalation: false

​          capabilities: {}

​          privileged: false

​          readOnlyRootFilesystem: false

​          runAsNonRoot: false

​        stdin: true

​        terminationMessagePath: /dev/termination-log

​        terminationMessagePolicy: File

​        tty: true

​        volumeMounts:

​        \- mountPath: /var/jenkins_home

​          name: vol1

​        \- mountPath: /etc/localtime

​          name: vol2

​        \- mountPath: /etc/timezone

​          name: vol3

​          subPath: timezone

​      dnsPolicy: ClusterFirst

​      hostAliases:

​      \- hostnames:

​        \- reg.ecs.local

​        ip: 10.191.129.56

​      imagePullSecrets:

​      \- name: harbor-name

​      restartPolicy: Always

​      schedulerName: default-scheduler

​      securityContext: {}

​      serviceAccount: jenkins-ci

​      serviceAccountName: jenkins-ci

​      terminationGracePeriodSeconds: 30

​      volumes:

​      \- name: vol1

​        persistentVolumeClaim:

​          claimName: ecs-jenkins

​      \- name: vol2

​        persistentVolumeClaim:

​          claimName: localtime

​      \- configMap:

​          defaultMode: 429

​          items:

​          \- key: timeZone

​            path: timezone

​          name: timezone

​          optional: false

​        name: vol3