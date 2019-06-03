## 1、升级devops前端的指令

```bash
kubectl get pods -o wide --all-namespaces 10.145.208.200；10.145.208.199		#

#查看devops的前端是否存在
cd  /app/container/paas-deploy/paas-k8s-deploy/paas-devops-fe
vim vim paas-svc-k8s.yaml
#更改后 kubectl apply -f paas-svc-k8s.yaml
#再次查看pods状态
#拉代码推代码的步骤
#先将外网代码down下来，然后分别进入代码文件夹
cd paas-ce-workload  git remote add shtel http://10.145.196.76:9080/Shtel-PaaS-DevOps/paas-devops-pipeline git push shtel "refs/remotes/origin/*:refs/heads/*" --tags 
```

## 2、常用命令总结

```bash
kubectl get svc --all-namespaces -o wide |grep 31187	#查看端口号是否被占用
rabbitmq-server -detached	#重启rabbitMQ 

# 查看所有pod  jvm信息
kubectl get pod --all-namespaces -o wide | grep '^prod' | awk '{system("kubectl describe pod -n "$1" "$2" | grep -c JAVA_OPTS >/dev/null || echo "$1" "$2" ")}' 

#查看pod信息，在master主机上执行
kubectl get pod -o wide -n 集群空间名 
kubectl exec 容器pod名 -n 集群空间名 -it "bash"		#解释：kubectl exec  pods 进入容器命令如下命令
kubectl exec uam-unifiedaccount-ws-b65fdb6-grpdn -n pe-uam-app -it /bin/bash 

#拉取引擎日志命令
kubectl cp  集群空间名/容器pod名:/usr/local/gateway-service/log/PaasMainApp_all.2019-04-17.2.log ./ 
# 解释：kubectl cp 引擎名称+pods名称:+路径 拷贝到当前目录，如下：
kubectl cp  pe-uam-app/uam-unifiedaccount-scheduled-55b84448bc-twrfx:/usr/local/logs/unifiedaccount ./ 

# 删除某个pod
kubectl delete pod -n te-acct acct-biz-bill-6778b46dd4-x9dgr

kubectl get pod -n te-pyz-app		#查看pod信息
kubectl get rs -n te-pyz-app		#查看资源
kubectl delete rs -n te-pyz-app 容器名		#删除资源pod
```



使用put 关闭实例：

http://10.145.167.36:30951/eureka/apps/CUSBIZVIEW/cus-biz-view--2c70e5e3-857c757bc4-m5l5n:cusBizView:9801/status?value=OUT_OF_SERVICE

使用put开启实例：

http://10.145.167.36:30951/eureka/apps/CUSBIZVIEW/cus-biz-view--2c70e5e3-857c757bc4-m5l5n:cusBizView:9801/status?value=UP





