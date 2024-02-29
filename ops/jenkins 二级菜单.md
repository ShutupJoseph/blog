#### Active Choices  插件
Active Choices Reactive Parameter 可以通过 Referenced parameters 里指定的变量去做动态刷新

#### 通过选择branch 来展示对应的commit
参数化构建Active Choices Parameter,Name branch,groovy:  
获取分支 通过jenkins内置方法  
groovy:  
```
import jenkins.*
import jenkins.model.* 
import hudson.*
import hudson.model.*


def jenkinsCredentials = com.cloudbees.plugins.credentials.CredentialsProvider.lookupCredentials(
        com.cloudbees.plugins.credentials.Credentials.class,
        Jenkins.instance,
        null,
        null
);
for (creds in jenkinsCredentials) {
    if(creds.id == "ab73b44b-a6bc-4e20-a5e3-63ab1c3e4dc5"){
         USER = creds.username
         PASS = creds.password
         def gettags = ("/usr/bin/git ls-remote -h http://$USER:$PASS@<git地址.git>").execute()
         return gettags.text.readLines().collect { 
             it.split()[1].replaceAll('refs/heads/', '').replaceAll('refs/tags/', '').replaceAll("\\^\\{\\}", '')
         }
     }
 }
```

参数化构建 Active Choices Reactive Parameter,Name commit,groovy  
Referenced parameters 指定 branch 这样就能做到 变换branch 来获取对应分支的commit  
获取 commit 后面是python写的fastapi接口 调用的gitlab api 这个功能并不难  
groovy:  
```
import groovy.json.*


def gitlab_token = "<gitlab token>"
def git_project = "<java/code>"
def git_domain = "<http://xxx.xxx.xxx>"

def conn = new URL("http://127.0.0.1:8000/api/commit_id?gitlab_token=${gitlab_token}&git_branch=${branch}&git_project=${git_project}&git_domain=${git_domain}").openConnection()
conn.setRequestMethod("GET")
def json = new JsonSlurper()
def result = json.parseText(conn.content.text)

def list = ''
for (value in result.data) {
    list += "${value.commit_id}  ${value.committer}  ${value.created_at}    ${value.branch}    ${value.message}"
}

return list.readLines()
```

构建用pipline:  
```
def project_list = "${project}".split(",") // project is Extended Choice Parameter name
def platform_list = "${platform}".split(",") // platform is Extended Choice Parameter name
def harbor_url = "<docker.xx.com/ops/>"  // docker仓库
def commit = "$commit".split()[0] // 获取到的commit是字符串(commitid commit信息 提交人 时间)类似格式

def kube_config = [
    "xx1": "config.xx1",
    "xx2": "config.xx2",
]


pipeline {
    agent any

    stages {
        stage('git pull') {
            steps {
                checkout scmGit(branches: [[name: "${commit}"]], extensions: [], userRemoteConfigs: [[credentialsId: '<ab73b44b-xxx>', url: 'http://<git域名>']])
            }
        }
        
        stage('close logfile') {
            steps {
                sh """cat > logback.xml << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE xml>
<configuration scan="true">
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <charset>UTF-8</charset>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>

</configuration>
EOF"""
                sh 'cat logback.xml | tee **/src/main/resources/logback-spring-prod.xml'
                sh 'cat logback.xml | tee **/src/main/resources/logback-spring.xml'
            }
        }
        
        stage('mvn install') {
            steps {
                sh "/usr/local/java/apache-maven-3.5.2/bin/mvn clean install"
            }
        }
        
        stage('docker build') {
            steps {
                script {
                    tagdate = new Date().format("yyyyMMddHHmmss",TimeZone.getTimeZone('Asia/Shanghai'))
                    commit = sh (script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.tag_ = "${tagdate}-${branch}-${commit}"
                    for (value in project_list) {
                        lower_value = value.toLowerCase()
                        sh "docker build -f ${value}/Dockerfile -t ${harbor_url}${lower_value}:${tag_} ${value}/target/"
                        sh "docker push ${harbor_url}${lower_value}:${tag_}"
                    }
                }
            }
        }
        stage('push to k8s') {
            steps {
                script {
                    for (value_pro in project_list) {
                        for (value_pla in platform_list) {
                            config = kube_config[value_pla]
                            lower_project_name = value_pro.toLowerCase()
                            sh "kubectl --kubeconfig ~/.kube/${config} set image  deployment/${lower_project_name} ${lower_project_name}=${harbor_url}${lower_project_name}:${tag_} -n ${value_pla}"
                        }
                    }
                    
                }
            }
        }
        
    }
}
```

