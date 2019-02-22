## jenkins rpm
1. 更新CentOS7系统
```
sudo yum install epel-release nodejs
sudo yum update
```
2. 安装java
```
sudo yum install java-1.8.0-openjdk.x86_64
```
3. 安装jenkins
```
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo yum install -y jenkins
```
4. 启动jenkins，并查看状态
```
sudo systemctl start jenkins
sudo systemctl status jenkins
```

## openvpn 安装
1. 安装openvpn
```
yum -y install openvpn
```
2. 拷贝配置文件到/etc/openvpn/config
```
配置文件如下：
ca.crt, ta.key, vpnzhub.crt, vpnzhub.csr, vpnzhub.key, zhub.ovpn
```
3. 启动openvpn
```
cd /etc/openvpn/config && openvpn zhub.ovpn &
```

## jenkins docker
1. 安装docker
```
sudo yum -y install yum-utils device-mapper-persistent-data lvm2 container-selinux libcgroup pigz libseccomp
sudo rpm -ivh libltdl7-2.4.2-alt7.x86_64.rpm
sudo rpm -ivh docker-ce-18.03.1.ce-1.el7.centos.x86_64.rpm
sudo curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://04be47cf.m.daocloud.io
```
2. 启动docker，并赋予jenkins用户操作docker的权限
```
sudo systemctl start docker
sudo usermod -a -G docker jenkins
sudo systemctl restart jenkins
```

## jenkins kubernetes


## jenkins操作
参考：https://jenkins.io/zh/doc/book/pipeline/syntax/

下载单个文件：  
`curl -O https://raw.githubusercontent.com/zbCodeLive/kubernetes/master/test/dockerfile`

### jenkins-docker(pipeline)
#### docker
1. docker pull
```
pipeline {
    agent any
    stages {
        stage('docker version') {
            steps {
                sh 'docker --version'
            }
        }
        stage('docekr pull') {
            steps {
                sh 'docker pull nginx'
            }
        }
    }
}
```
2. docker pull and start
```
拉取harbor上的镜像，并进行操作
pipeline {
    agent { 
        docker {
            registryUrl 'https://zhub.com'
            registryCredentialsId 'docker-id'
            image 'zhub.com/centos/centos-java'
        }
    } 
    stages {
        stage('javaVersion') {
            steps {
                sh 'tail -f /etc/hostname'
            }
        }
    }
}
```
3. 在多个docker上执行命令
```
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent none 
    stages {
        stage('Example Build') {
            agent { docker 'maven:3-alpine' } 
            steps {
                echo 'Hello, Maven'
                sh 'mvn --version'
            }
        }
        stage('Example Test') {
            agent { docker 'openjdk:8-jre' } 
            steps {
                echo 'Hello, JDK'
                sh 'java -version'
            }
        }
    }
}
```

#### dockerfile
```
mkdir -p /data/DockerFile/nginx/
pipeline {
    agent any
    stages {
        stage('docker version') {
            steps {
                sh 'docker --version'
            }
        }
        stage('docekrfile') {
            steps {
                sh 'mkdir -p /data/DockerFile/nginx/'
                sh 'echo -e "docker\ndockerfile" > /data/DockerFile/nginx/dockerfile'
                sh """
                cat > /data/DockerFile/nginx/ca-config.json <<EOF
                {
                  "signing": {
                    "default": {
                      "expiry": "87600h"
                    },
                    "profiles": {
                      "www": {
                         "expiry": "87600h",
                         "usages": [
                            "signing",
                            "key encipherment",
                            "server auth",
                            "client auth"
                        ]
                      }
                    }
                  }
                }
                EOF
                """
            }
        }
    }
}

ps:建议不要在pipeline中写长文本，影响代码美观，且写入后会有许多的空格
```

```
使用github服务器上的dockerfile文件
pipeline {
    agent {label 'slave-1'}
    stages {
        stage('get dockerfile') {
            steps {
                git 'https://github.com/zbCodeLive/kubernetes'   
            }
        }
        stage('docker build') {
            steps {
                sh 'cd test && docker build -t testdocker .'
            }
        }
    }
}
```


### jenkins-kubernetes(pipeline)
配置jenkins服务器与kubernetes的认证(此为https认证)  
```
1. 拷贝kubectl到jenkins服务器上
scp kubectl root@192.168.12.248:/opt/kubernetes/bin

2. 拷贝认证文件到jenkins服务器上
scp ca.pem admin.pem admin-key.pem root@192.168.12.248:/root

3. 在jenkins服务器上生成～/.kube/config
kubectl config set-cluster kubernetes --server=https://192.168.1.241:6443 --certificate-authority=/root/ca.pem
kubectl config set-credentials cluster-admin --certificate-authority=/root/ca.pem --client-key=/root/admin-key.pem --client-certificate=/root/admin.pem
kubectl config set-context default --cluster=kubernetes --user=cluster-admin
kubectl config use-context default

4. 查看～/.kube/config文件
[root@localhost ~]# cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /root/ca.pem
    server: https://192.168.1.241:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: cluster-admin
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: cluster-admin
  user:
    client-certificate: /root/admin.pem
    client-key: /root/admin-key.pem

5. 验证kubectl是否能够获取到指定集群的信息
[root@localhost ~]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6dbc5f58df-b9gtl   1/1     Running   2          50d
nginx-deployment-6dbc5f58df-gvmfx   1/1     Running   2          50d
nginx-deployment-6dbc5f58df-lfgfz   1/1     Running   0          47d
pv-pod                              1/1     Running   0          41d
pv-pod2                             1/1     Running   0          41d
```

### jenkins-slave
1. 本地安装slave
![image](https://note.youdao.com/yws/public/resource/ea5497befa13efb68712c062bdee2d37/xmlnote/8FB16F2F804C47E88B7F4AB3ABCF1F5E/2760)
2. 远程安装salve
![image](https://note.youdao.com/yws/public/resource/ea5497befa13efb68712c062bdee2d37/xmlnote/51BE3933F6824623AC7AC0D194F4B671/2765)


### jenkins-pipeline-post
根据结果进行操作
```
Conditions
always
无论流水线或阶段的完成状态如何，都允许在 post 部分运行该步骤。

changed
只有当前流水线或阶段的完成状态与它之前的运行不同时，才允许在 post 部分运行该步骤。

failure
只有当前流水线或阶段的完成状态为"failure"，才允许在 post 部分运行该步骤, 通常web UI是红色。

success
只有当前流水线或阶段的完成状态为"success"，才允许在 post 部分运行该步骤, 通常web UI是蓝色或绿色。

unstable
只有当前流水线或阶段的完成状态为"unstable"，才允许在 post 部分运行该步骤, 通常由于测试失败,代码违规等造成。通常web UI是黄色。

aborted
只有当前流水线或阶段的完成状态为"aborted"，才允许在 post 部分运行该步骤, 通常由于流水线被手动的aborted。通常web UI是灰色。
```
```
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
    post { 
        always { 
            echo 'I will always say Hello again!'
        }
    }
}
按照惯例, post 部分应该放在流水线的底部。
Post-condition 块包含与 steps 部分相同的steps
```