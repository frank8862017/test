pipeline {
    agent any

    environment {
       BUILD_USER_EMAIL = "xx@xx.com"
       BUILD_USER = "testit"
    }

    parameters {
        string(name:'pomPath', defaultValue: 'pom.xml', description: 'pom.xml的相对路径')
        string(name:'lineCoverage', defaultValue: '50', description: '单元测试代码覆盖率要求(%)，小于此值pipeline将会失败！')
    }

    //pipeline运行结果通知给触发者
    post{
        success{
            script {
                wrap([$class: 'BuildUser']) {
                    mail to: "${BUILD_USER_EMAIL}",
                            subject: "PineLine '${JOB_NAME}' (${BUILD_NUMBER}) result",
                            body: "${BUILD_USER}'s pineline '${JOB_NAME}' (${BUILD_NUMBER}) run success\n请及时前往${env.BUILD_URL}进行查看"
                }
            }
        }
        failure{
            script {
                wrap([$class: 'BuildUser']) {
                    mail to: "${BUILD_USER_EMAIL}",
                            subject: "PineLine '${JOB_NAME}' (${BUILD_NUMBER}) result",
                            body: "${BUILD_USER}'s pineline  '${JOB_NAME}' (${BUILD_NUMBER}) run failure\n请及时前往${env.BUILD_URL}进行查看"
                }
            }

        }
        unstable{
            script {
                wrap([$class: 'BuildUser']) {
                    mail to: "${BUILD_USER_EMAIL}",
                            subject: "PineLine '${JOB_NAME}' (${BUILD_NUMBER})结果",
                            body: "${BUILD_USER}'s pineline '${JOB_NAME}' (${BUILD_NUMBER}) run unstable\n请及时前往${env.BUILD_URL}进行查看"
                }
            }
        }
        aborted{
            script {
                wrap([$class: 'BuildUser']) {
                    echo 'I will always say Hello again!'
                    mail to: "${BUILD_USER_EMAIL}",
                            subject: "PineLine '${JOB_NAME}' (${BUILD_NUMBER})结果",
                            body: "${BUILD_USER}'s pineline '${JOB_NAME}' (${BUILD_NUMBER}) run unstable\n请及时前往${env.BUILD_URL}进行查看"
                }
            }
        }
    }

    stages {
        stage('单元测试') {
           when { anyOf{branch 'ci';branch 'testbranch1.5'} }
           steps {
               echo "starting unitTest......"
               echo "BUILD_USER_EMAIL:${BUILD_USER_EMAIL }"
               echo "JOB_NAME:${JOB_NAME }"
               echo "BUILD_NUMBER:${BUILD_NUMBER }"
               echo "BUILD_USER:${BUILD_USER }"
               echo "JOB_NAME:${JOB_NAME }"
               echo "BUILD_NUMBER:${BUILD_NUMBER }"
               //注入jacoco插件配置,clean test执行单元测试代码. All tests should pass.
            sh "mvn clean"
               sh "mvn org.jacoco:jacoco-maven-plugin:prepare-agent -f ${params.pomPath} clean test -Dautoconfig.skip=true -Dmaven.test.skip=false -Dmaven.test.failure.ignore=true"
               junit '**/target/surefire-reports/*.xml'
               //配置单元测试覆盖率要求，未达到要求pipeline将会fail,code coverage.LineCoverage>20%.
               jacoco changeBuildStatus: true, maximumLineCoverage:"${params.lineCoverage}"
           }
        }

        stage('静态检查') {
           when { anyOf{branch 'ci';branch 'testbranch1.5'} }
           steps {
               echo "starting codeAnalyze with SonarQube......"
               //sonar:sonar.QualityGate should pass
               withSonarQubeEnv('sonarserver') {
                   //固定使用项目根目录${basedir}下的pom.xml进行代码检查
                   //sh "/usr/local/sonar-scanner/bin/sonar-scanner "
                   sh '''dirSrc=`ls -m $WORKSPACE/**/target/jacoco.exec|tr -d " "`
                mvn -f pom.xml sonar:sonar -Dsonar.login=rui.li -Dsonar.password=123456 -Dsonar.core.codeCoveragePlugin=jacoco -Dsonar.exclusions="**/bean/**,**/entity/**" -Dsonar.jacoco.reportPaths="$dirSrc" -Dsonar.dynamicAnalysis=reuseReports'''
               }
               script {
                   timeout(2) {
                       //利用sonar webhook功能通知pipeline代 码检测结果，未通过质量阈，pipeline将会fail
                       def qg = waitForQualityGate()
                       echo "${qg.status}"
                       if (qg.status != 'OK') {
                           echo "${qg.status}"
                           error "未通过Sonarqube的代码质量阈检查，请及时修改！failure: ${qg.status}"
                       }
                   }
               }
           }
        }

        stage('Deploy UAT for test-web'){
            when {branch 'testbranch1.5'}
            steps{
                timeout(time: 5, unit: 'MINUTES'){
                    input message:'Do you approve deploy UAT for test-web?'

                    sh "chmod +x  ./test-web/Package.sh"
                    sh "sh -x ./test-web/Package.sh Package uat"
                    sh "sh -x ./test-web/Package.sh Publish"
                    sh " sed -i 's/test-web:latest/test-web:${GIT_COMMIT.substring(0,7)}/g'  ./test-web/kubernetes/kubernetes-config-uat.yaml "
                    sh " kubectl apply -f ./test-web/kubernetes/kubernetes-config-uat.yaml "
                }
            }
        }

        stage('Deploy UAT for test-task'){
            when {branch 'testbranch1.5'}
            steps{
                timeout(time: 5, unit: 'MINUTES'){
                    input message:'Do you approve deploy UAT for test-task?'

                    sh "chmod +x  ./test-task/Package.sh"
                    sh "sh -x ./test-task/Package.sh Package uat"
                    sh "sh -x ./test-task/Package.sh Publish"
                    sh " sed -i 's/test-task:latest/test-task:${GIT_COMMIT.substring(0,7)}/g'  ./test-task/kubernetes/kubernetes-config-uat.yaml "
                    sh " kubectl apply -f ./test-task/kubernetes/kubernetes-config-uat.yaml "
                }
            }
        }

        stage('Deploy PRO for test-web'){
            when {branch 'testbranch1.5'}
            steps{
                timeout(time: 5, unit: 'MINUTES'){
                    input message:'Do you approve deploy PRO for test-web?'

                    sh "chmod +x  ./test-web/Package.sh"
                    sh "sh -x ./test-web/Package.sh Package pro"
                    sh "sh -x ./test-web/Package.sh Publish"
                    sh " sed -i 's/test-web:latest/test-web:${GIT_COMMIT.substring(0,7)}/g'  ./test-web/kubernetes/kubernetes-config-pro.yaml "
                    sh " kubectl apply -f ./test-web/kubernetes/kubernetes-config-pro.yaml "
                }
            }
        }

        stage('Deploy PRO for test-task'){
                    when {branch 'testbranch1.5'}
                    steps{
                        timeout(time: 5, unit: 'MINUTES'){
                            input message:'Do you approve deploy PRO for test-task?'

                            sh "chmod +x  ./test-task/Package.sh"
                            sh "sh -x ./test-task/Package.sh Package pro"
                            sh "sh -x ./test-task/Package.sh Publish"
                            sh " sed -i 's/test-task:latest/test-task:${GIT_COMMIT.substring(0,7)}/g'  ./test-task/kubernetes/kubernetes-config-pro.yaml "
                            sh " kubectl apply -f ./test-task/kubernetes/kubernetes-config-pro.yaml "
                        }
                    }
                }
    }
}
