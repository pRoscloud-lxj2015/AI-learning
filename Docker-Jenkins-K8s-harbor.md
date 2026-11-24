# 本地部署安装

环境配置：
系统：windows 10,  wsl2_ubuntu24.04
内存：32G，500G硬盘

## docker 
下载：[Docker-Desktop:v4.99.0](https://docs.docker.com/desktop/setup/install/windows-install/)

在安装文件所在目录 管理员打开PowerShell
```
Start-Process 'Docker Desktop Installer.exe' -Wait -ArgumentList \
   'install', '--installation-dir="安装目录\Docker"', '--accept-license'
```
汉化版：必须选择和[Docker-desktop](https://github.com/asxez/DockerDesktop-CN)版本对应

设置Setting-> Docker 引擎：添加以下镜像源地址：
```
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://mtzjcnqh.mirror.aliyuncs.com",
    "https://docker.m.daocloud.io",
    "https://gcr.m.daocloud.io",
    "https://k8s.m.daocloud.io",
    "https://dockerproxy.net"
  ]
```
由于安装 磁盘镜像位置默认在C盘。需要迁移出来：
```
设置 -> Resources(资源) -> 高级 选择自己的放置 路径 
```
最后点击**应用并重启**

## Jenkins
安装
```
# 在wsl_ubuntu 执行的命令： 宿主机就是Ubuntu。也可以在PowerShell下执行，宿主机就是Windows
# 加 -rm 则退出时，会自动删除创建的容器
docker run -u root -d --name jenkins-blueocean -p 8080:8080 -p 50000:50000  \
   -e JENKINS_JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true" \
   -v /mnt/c/Users/lxjfr2017:/home -v jenkins-data:/var/jenkins_home \
   -v /var/run/docker.sock:/var/run/docker.sock -v /usr/bin/docker:/usr/bin/docker \
   -v /usr/local/lib/docker/cli-plugins:/usr/local/lib/docker/cli-plugins \
   -e TZ="Asia/Shanghai" -e JAVA_OPTS="-Duser.timezone=Asia/Shanghai" \
   jenkinsci/blueocean
   jenkins/jenkins:lts  或者：jenkinsci/blueocean
   
  # - 使用jenkinsci/blueocean 进入容器，显示固定的值修改方法： echo 'export PS1="[\u@\h \w]\$ "' >> ~/.bashrc
   
  # -v /usr/bin/docker:/usr/bin/docker 是为了直接使用Ubuntu下的Docker。不想在jenKins单独安装Docker
   
  # -v /mnt/c/Users/lxjfr2017:/home 将前者挂载到jenkins主机上。但这会出现一个问题：
     构建Pipeline时 使用本地Github仓库时，无法访问到这路径下的权限
# 查看初始密码命令：
   docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword 

出现一串数字，说明成功。
1.修改镜像源：
  docker exec -it (name)jenkins-blueocean bash    ->  访问容器
   cd /var/jenkins_home/updates/    注意，Jenkins 是在这var目录下
  cp default.json default.json.bak  备份
  sed -i 's|https://updates.jenkins.io/download|https://mirrors.tuna.tsinghua.edu.cn/jenkins|g' default.json
  sed -i 's|https://updates.jenkins-ci.org/download|https://mirrors.tuna.tsinghua.edu.cn/jenkins|g' default.json
  sed -i 's|www.google.com|www.baidu.com|g' default.json
2.修改整机镜像源： /etc/apt/sources.list.d/debian.sources   对应：debian 13版本
    URIs: https://mirrors.tuna.tsinghua.edu.cn/debian
    Components: main contrib
    
    URIs: https://mirrors.tuna.tsinghua.edu.cn/debian-security
    Components: main contrib
    保存：执行apt updates 更新和验证。
    systemctl restart jenkins 重启jenkins
3. 安装pytest : pip3 install pytest --break-system-packages
```
浏览地址：[]()http://localhost:8080

### 推送到 Harbor
在Jenkins的 设置中，全局凭据中添加 全局凭据 -> 选全局,填入Harbor的 用户名， 密码。ID 设置harbor_credentials
```
Jenkinsfile使用：
 steps{
    withCredentials([
        uernamePassword(credentialsId: '', usernameVariable: 'name', usernameVariable: 'password')
    ]){
        sh '执行 echo 'password' | docker login xxx ' docker push 等
    }
 }
```

### 部署K8s
1. 配置全局凭据： setting -> 凭据管理 -> 全局凭据 -> 选 PKCS12 凭据文件：密码：填生成时带的密码
```
进入minikube 生成的 .minikue目录下profiles\minikube；执行下面命令：生成.pfx
openssl pkcs12 -export -out minikube.pfx -inkey apiserver.key -in apiserver.crt -certfile ../../ca.crt -passout pass:123456
```
```
steps{
    withKubeConfig([credentialsId: 'minikube_cretdentials', serverUrl: 'https://minikube ip获得:8443']){
        sh '执行kubectl set image deployment/**  -n namespace 等'
    }
}
```
2. 使用云端节点来，根据yaml 来给k8s创建节点.(可以配置私有仓库Harbor拉取镜像)。安装Kubernetes 插件

   配置全局凭据： setting -> Clouds -> New cloud -> Type选Kubernetes -> K8s地址(minikube ip获取)端口8443 / K8s Server
   证书key(minikube\.minikube目录下的ca.crt 全部拷贝填) ，点击连接测试,出现Connected to Kubernetes v1.xx.xx 表示成功。
```
pipeline使用：
agent{
   kubernetes{
      cloud 'cloud_name',
      yaml '''
      
      '''
   }
}
配置私有仓库：需要创建Harbor的Secret：
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.example.com \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  -n mynamespace
创建完后，在yaml ,通过 imagePullSecrets引用：harbor-secret
imagePullSecrets:
- name: harbor-secret
```

## Harbor
必备的条件

|  软件 | 版本  | 描述  |
|---|---|---|
|  Docker 引擎 | 版本 > 20.10  | [Docker 引擎安装](https://docs.container.net.cn/engine/install/) |
| Docker Compose  | Docker Compose > 2.3  |  	Docker Compose 是 Docker 引擎的一部分  |
| OpenSSL(1.1.1q) | 最新 (可选) | 用于为 Harbor 生成证书和密钥 |

下载：[Harbor:v2.15.5](https://github.com/goharbor/harbor/releases/tag/v2.15.5) 离线版本
```
wget https://gh-proxy.org/https://github.com/goharbor/harbor/releases/download/v2.15.5/harbor-offline-installer-v2.15.5.tgz
下载慢，可用这个来替代
```

将离线包放置到 wsl-ubuntu下的目录(~/)：一定要注意文件权限: ls -l
```
cp /mnt/**.v2.15.5.tgz ~/*
tar xzvf harbor-online-installer-version.tgz
待解压完，放置在harbor目录下
```
### openssl 生成证书
在这里配置https 访问。生产环境使用 *CA 提供商* 例如[Let’s Encrypt](https://letsencrypt.openssl.ac.cn/)
```
cd harbor_data_cert
```
yourdomain.com: lxjfr.harbor.com
IP: 宿主机的IP：**hostname -I** 查看

生成证书颁发机构证书：

1.生成 **ca证书私钥**
```
openssl genrsa -out ca.key 4096
```
2.生成 **CA 证书**：
```
openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal \
   /CN=lxjfr.harbor.com_ca" -key ca.key -out ca.crt
注意参数: CN 和服务器证书的CN 要区别开来，这里使用yourdomain.com_ca
```

生成服务器证书:

1.生成**私钥**:
```
openssl genrsa -out yourdomain.com.key 4096
```
2.生成证书签名请求 (CSR)
```
openssl req -sha512 -new -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
    -key yourdomain.com.key -out yourdomain.com.csr
```
3.生成 x509 v3 扩展文件:
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS=yourdomain.com
IP=**
多个域名和IP使用： DNS.1= **, IP.1=** 
```
4.使用 v3.ext 文件为您的 Harbor 主机生成证书
```
openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA ca.crt -CAkey ca.key -CAcreateserial \ 
   -in yourdomain.com.csr -out yourdomain.com.crt
```
检查证书是否完整：
1.检查SAN：
```
openssl x509 -in yourdomain.com.crt -noout -text | grep -A 1 "Subject Alternative Name" 验证SAN是否yourdomain
openssl verify -CAfile ca.crt yourdomain.com.crt  返回OK
```
提供给Harbor主机的证书：
```
yourdomain.com.crt
yourdomain.com.key
```
提供给Docker 的证书：放置 **etc/docker/certs.d/yourdomain.com/** 目录下
```
openssl x509 -inform PEM -in yourdomain.com.crt -out yourdomain.com.cert
放置在 Docker主机下：/etc/docker/certs.d/yourdomain.com
yourdomain.com.cert 
yourdomain.com.key
ca.crt
```
### 配置TLS 组件之间的通信的证书生成
所需要的证书 通过 **prepare**生成
```
在 Ubuntu下 执行
docker run -v /:/hostfs goharbor/prepare:<current_harbor_version> gencert -p /path/to/internal/tls/cert
/:/hostfs  是挂载所在的目录
<current_harbor_version> :所使用的Harbor的版本号。这里使用的版本：V.2.15.5
```

### 安装Harbor：
cd harbor下，修改 harbor.yml
```
hostname: yourdomain.com  # reg.mydomain.com

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80

# https related config
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /harbor_data_cert/yourdomain.com.crt 绝对路径 # your/certificate/path
  private_key: /harbor_data_cert/yourdomain.com.key 绝对路径  # /your/private/key/path
  # enable strong ssl ciphers (default: false)
  # strong_ssl_ciphers: false
internal_tls:
#   # set enabled to true means internal tls is enabled
   enabled: true
#   # put your cert and key files on dir
   dir: harbor_data_cert/internal/tls/cert # /etc/harbor/tls/internal
```
执行 **sudo ./install.sh**  最后看到 success  表示成功
现在主机下：测试是否连通
```
返回 Pong 则成功
curl -k https://yourdomain.com/api/v2.0/ping 
curl -k https://IP/api/v2.0/ping 
有返回， 则成功
curl -v -k https://yourdomain.com/api/v2.0/health
curl -v -k https://IP/api/v2.0/health
```

### docker login 登录：
在 Docker-desktop下添加(直接绕过访问)：
```
"insecure-registries": [
    "lxjfr.harbor.com",
    "172.25.171.102"
]
```


常见问题：
1. IP能访问，域名无法访问： 通常是DNS解析失败所致。
```
Ubuntu  /etc/hosts  添加： IP  yourdomain.com
Window10 C:\Windows\System32\drivers\etc\HOSTS  IP  yourdomain.com
```
2. JobService 一直Restarting的问题： 查看日志 是因为 加载 harbor_curs_ca.crt找不到
```
将该证书 cp internal/tls/cert/harbor_internal_ca.crt ~/harbor/common/config/shared/trust-certificates
```
3.修改 **harbor.yml** 后
```
执行顺序：.prepare 
docker compose down -v  不会删除数据
sudo docker compose up -d   重新创建
启动：
docker compose start / stop (不建议使用restart 不太好使)
```
4.删除干净：
```
docker compose down -v
sudo docker rmi $(sudo docker images --filter=reference='goharbor/*' -q)

删除以下目录释放存储空间：
    /data：数据库、Redis、镜像存储（重装必删）。
    /etc/harbor：配置文件（建议删除）。
    /var/log/harbor：日志文件（可选）。
```
参考[Harbor官网文档](https://goharbor.cn/docs/2.13.0/install-config/download-installer/)

## K8s
注意单独下载 [kubectl](https://dl.k8s.io/release/v1.37.0/bin/windows/amd64/kubectl.exe)  需要连 校验文件一起下载

### 使用 **minikube**单点管理集群
下载：[minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2Fwindows+package+manager)
```
安装命令 PowerShell进入到minikube.exe文件目录下打开：
  $env:Path += ';D:\minikube'; [Environment]::SetEnvironmentVariable('Path', $env:Path, 'Machine')

```
1.打开PowerShell执行
```
minikube start --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.39.0.iso \
  --base-image='registry.cn-hangzhou.aliyuncs.com/google_containers/kicbase:v0.0.48' --kubernetes-version=1.36.3 \
   --registry-mirror=https://registry.docker-cn.com --image-mirror-country=cn \
   --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
   --driver=docker --network-plugin=cni --cni=flannel --extra-config=kubelet.cgroup-driver=systemd
下载很慢：
可先下载 docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kicbase:v0.0.48
```
2.查看 minikube status：
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```
3.查看 minikube 是否 Ready 状态：**kubectl get node -o wide**  :    minikube   Ready    control-plane   27h   v1.36.3 
4. **minikube dashboard --alsologtostderr -v=1 --url**  成功会 得到 **URL** 访问地址
有错误的处理方法：
```
1.查看pod节点信息： kubectl get pod -A / minikube kubectl get pod -A  全部Running说明没问题
  查看指定pod的状态：kubectl get pods -n kubernetes-dashboard -o wide
2.ImagePullBackOFF: 拉取镜像出问题：
   - 查看deployment下的镜像文件：kubectl get deployment -n kubernetes-dashboard -o yaml | grep image:
     或者直接 docker images 
   - 出现无法加载或者没有： 使用  docker pull 地址/对应版本： 拉取  地址看下面有对应的替换就行
   - 使用下面的命令，查看具体pod 的详细日志：
      kubectl describe pod NAME -n NAMESPACE   -> 使用 kubectl get pod -A 查看 NAMESPACE 和 NAME
   - 缺失的使用： docker pull *** 
     docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/dashboard:v2.7.0
     docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-scraper:v1.0.8
   - 编辑文件，修改镜像加载地址：
     kubectl edit deployment NAME -n NAMESPACE       NAME(后面的数字ID忽略)  
     修改两处的 Images： docker images 得到就行
   - 会自动去尝试拉取，全部成功后，回全现实 Running 状态。之后，执行 
      minikube dashboard --alsologtostderr -v=1 --url
3.还可以使用 minikube ssh 主机内部去拉取，但不建议
  - sudo systemctl status docker.service   查看docker运行状态
    sudo systemctl unmask docker.service   擦除docker  mask 状态
    sudo systemctl start docker.service   启动 docker服务
  - docker pull ****
    docker tag sources  target 打标签。不会复制镜像文件，只是个标签。
  exit 退出
```

常用镜像文件的替换地址：
```
gcr.io	           gcr.m.daocloud.io	    人工配置，新源站可提 Issue 申请
k8s.gcr.io	       k8s-gcr.m.daocloud.io	已迁移到 registry.k8s.io
registry.k8s.io	   k8s.m.daocloud.io	
docker.io	       docker.m.daocloud.io
docker.1ms.run
dockerproxy.net  /qnib/pytest
```
参考[K8s官网](https://kubernetes.io/zh-cn/docs/home/)
参考[minikube文档](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)