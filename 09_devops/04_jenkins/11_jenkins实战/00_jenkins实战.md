# jenkins实战

## 一、实战一

```yaml
#!groovy

@Library('jenkins-shared-library@main') _

def checkout = new org.checkout()
def tools = new org.color()

pipeline {

	agent { 
		node { 
			label "slave"
		}
	}

    options {
      timestamps()      
      parallelsAlwaysFailFast()
      timeout(time: 600, unit: 'SECONDS') 
      disableConcurrentBuilds(abortPrevious: true) 
      buildDiscarder(logRotator(numToKeepStr: '30'))
      skipDefaultCheckout() 
    }

    parameters {
        extendedChoice defaultValue: 'main', description: '构建分支', descriptionPropertyValue: 'main,master', multiSelectDelimiter: ',', name: 'build_branch',
        quoteValue: false, saveJSONParameterToFile: false, type: 'PT_RADIO', value: 'main,master', visibleItemCount: 5        
    }

    environment {
        String year = new Date().format("yyyy") 
        String month = new Date().format("MMdd") 
        String day = new Date().format("HHmm") 
        String second = new Date().format("ss")
		image_prefix = "registry.cn-hangzhou.aliyuncs.com/tool-bucket/tool"   
        giturl = "http://10.0.7.30/golang/go.git"     
    }

    stages {
        stage('1. 拉取代码') {
            steps {
                script {
                    cleanWs()
                    // 需要安装ansiColor插件
                    tools.PrintMessage("1.拉取代码","blue")
                    checkout.scm(build_branch,giturl)
                }
            }
        }
        stage('2. 构建镜像') {             
            steps {
                script {                    
                    tools.PrintMessage("2.构建镜像","blue")
                    try{       
                        ImageTag = "${image_prefix}:${BUILD_USER}-${build_branch}-${year}${month}${day}${second}-${BUILD_TAG}"
                        sh """
                            docker run --rm \
                                -v `pwd`:/workspace \
                                -v /root/.docker/config.json:/kaniko/.docker/config.json:ro \
                                gcr.io/kaniko-project/executor:latest \
                                --dockerfile=Dockerfile \
                                --destination=${ImageTag} \
                                --cache-copy-layers \
                                --cache=true \
                                --cache-repo=${image_prefix} \
                        """
                    }catch (error){
                        env.error = sh (returnStdout: true, script: "echo 第1步构建镜像失败：${error}").trim()
                        echo "Caught: ${error}"
                        sh "exit 1"
                    }
                }
            }
        }
        stage('3.部署') {
            steps {
                script {
                    tools.PrintMessage("3.部署","blue")
                    sh """
                        sed -i "s#image: .*#image: ${ImageTag}#" deploy.yaml
                        kubectl apply -f deploy.yaml
                    """
                }
            }
        }        
    }

    post {
        failure {
			script{
				sh "echo failure"
			}
        }
        success {
			script  {
              sh """
                curl -X POST -H "Content-Type: application/json" \
                        -d '
                        {
                        "msg_type": "post",
                        "content": {
                                "post": {
                                        "zh_cn": {
                                                "title": "Jenkins执行结束-xiaowu-${BUILD_TAG}项目",
                                                "content": 
                                                        [
                                                            [       
                                                                {
                                                                        "tag": "text",
                                                                        "text": "构建结果: "
                                                                },
                                                                {
                                                                        "tag": "text",
                                                                        "text": "成功-xiaowu"
                                                                }
                                                            ]
                                                        ]
                                                  }
                                         }
                                    }
                        }               
                ' \
                    https://open.feishu.cn/open-apis/bot/v2/hook/e117c9b1-1428-4393-85b4-4cf605304dbb											  
              """      
            }               
        }                                  
		unstable {
			script{
				sh "echo 构建不稳定"
			}
		}
		aborted {
			script{
				currentBuild.displayName = "${BUILD_NUMBER}次构建！构建取消!!!"
			}
		}
    }
}

```

## 二、静态代理实战

```yaml
#!groovy

@Library('jenkins-shared-library@main') _

def tools = new org.color()
def checkout = new org.checkout()

pipeline {

	agent {
        label 'slave'
    }

    options {
      timestamps()      
      parallelsAlwaysFailFast()
      timeout(time: 600, unit: 'SECONDS') 
      disableConcurrentBuilds(abortPrevious: true) 
      buildDiscarder(logRotator(numToKeepStr: '30'))
      skipDefaultCheckout() 
    }

    environment {
        String year = new Date().format("yyyy") 
        String month = new Date().format("MMdd") 
        String day = new Date().format("HHmm") 
        String second = new Date().format("ss")
		images_head = "registry.cn-hangzhou.aliyuncs.com"   
        giturl = "http://10.0.7.30/golang/go.git"           
    }

    parameters {
        choice choices: ['main', 'pre', 'test'], name: 'branch_name'
    }

    stages {
        stage('1.克隆代码') {
            agent {
                docker {
                    label 'slave'
                    image 'registry.cn-hangzhou.aliyuncs.com/tool-bucket/tool:git'
                }
            }   
            steps {
                script {
                    cleanWs() 
                    tools.PrintMessage("1.克隆代码","blue")   
                    checkout.scm(branch_name,giturl)
                }
            }
        }        
        stage('2.SonarQube扫描') {
            agent {
                docker {
                    label 'slave'
                    image 'sonarsource/sonar-scanner-cli'
                }
            }        
            steps {
                script {
                    tools.PrintMessage("2.SonarQube扫描","blue")              
                    withSonarQubeEnv('sonarqube') {
                        sh """
                        sonar-scanner \
                        -Dsonar.projectKey=test-go \
                        -Dsonar.projectName=test-go \
                        -Dsonar.projectVersion=test-go-${BUILD_NUMBER} \
                        -Dsonar.ws.timeout=30 \
                        -Dsonar.sources=. \
                        -Dsonar.sourceEncoding=UTF-8
                        sleep 3
                        """
                    }
                }  
            }
        }
        stage("3.质量门禁"){           
            steps {
                script {      
                    tools.PrintMessage("3.质量门禁","blue")
                    timeout(time: 10, unit: 'SECONDS') { 
                        def qg = waitForQualityGate('sonarqube') 
                        if (qg.status != 'OK') {
                            error "未通过Sonarqube的代码质量阈检查，请及时修改！failure: ${qg.status}"
                        }
                    }
                }
            }
        }         
        stage('4.构建镜像') {  
            agent {
                docker {
                    label 'slave'
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }                                                             
            steps {                
                script {                    
                    tools.PrintMessage("4.构建镜像","blue")
                    ImageTag = "${images_head}/tool-bucket/tool:${BUILD_USER}-${branch_name}-${year}${month}${day}${second}-${BUILD_TAG}"                     
                    docker.withRegistry("https://${images_head}", 'aliyun-images-registry') {
                        def customImage = docker.build("${ImageTag}")
                        customImage.push()                           
                    }                                            
                }                        
           
            }
        }     
        stage('5.部署') {
            agent {
                docker {
                    label 'slave'
                    image 'registry.cn-hangzhou.aliyuncs.com/tool-bucket/tool:kubectl'                                       
                    args '-v /root/.kube:/root/.kube'
                }
            }        
            steps {
                script {
                    tools.PrintMessage("5.部署","blue")
                    sh """
                        sed -i "s#image: .*#image: ${ImageTag}#" deploy.yaml
                        kubectl apply -f deploy.yaml
                    """
                }
            }
        }
        stage('6.MeterShpere接口测试') {
            steps {
                script {
                    tools.PrintMessage("6.MeterShpere接口测试","blue")
                    meterSphere method: 'testPlan',
                    mode: 'serial', 
                    msAccessKey: 'OYPWJwNk9vi6JNiF', 
                    msEndpoint: 'http://10.0.7.27:8081/', 
                    msSecretKey: '7D0UlTZXXTZMt0fD', 
                    openMode: 'auth', 
                    projectId: 'a6ad182f-f6f1-44de-bdc7-3e6aef7c9e84', 
                    projectName: '', 
                    projectType: 'projectId', 
                    resourcePoolId: '237d98d4-40b9-11ee-a07c-0242ac1e0a09', 
                    testCaseId: '', testCaseName: '', 
                    testPlanId: '99355526-981f-4927-bd6d-b37e54eb00c1', 
                    testPlanName: '', 
                    workspaceId: '4d144486-dc2a-4282-acd6-9b449869eeae'                    
                }
            }  
        }                     
    }
    post {
        failure {
			script{
               manager.addShortText("${BUILD_NUMBER}次构建! 构建失败!!!") 
			}
        }
        success {
			script  {
                manager.addShortText("${BUILD_NUMBER}次构建! 构建成功!!!")
            }               
        }                                  
		aborted {
			script{
				manager.addShortText("${BUILD_NUMBER}次构建!构建取消!!!")
			}
		}
		always {
            script{
			    manager.addShortText("构建用户: ${BUILD_USER}")
            }
		}	        
    }
}

```

## 三、动态代理实战

```bash
apiVersion: v1
kind: Secret
metadata:
  name: kubeconfig-secret
type: Opaque
data:
  #config: $(base64 -w 0 /root/.kube/config)
  config: YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci0gY2x1c3RlcjoKICAgIGNlcnRpZmljYXRlLWF1dGhvcml0eS1kYXRhOiBMUzB0TFMxQ1JVZEpUaUJEUlZKVVNVWkpRMEZVUlMwdExTMHRDazFKU1VSQ1ZFTkRRV1V5WjBGM1NVSkJaMGxKV1VoTlQyTm5iM0JFVlVGM1JGRlpTa3R2V2tsb2RtTk9RVkZGVEVKUlFYZEdWRVZVVFVKRlIwRXhWVVVLUVhoTlMyRXpWbWxhV0VwMVdsaFNiR042UVdWR2R6QjVUWHBGZUUxRVVYbE5SRTB4VGxST1lVWjNNSHBOZWtWNFRVUkZlVTFFVVhkT1ZFNWhUVUpWZUFwRmVrRlNRbWRPVmtKQlRWUkRiWFF4V1cxV2VXSnRWakJhV0UxM1oyZEZhVTFCTUVkRFUzRkhVMGxpTTBSUlJVSkJVVlZCUVRSSlFrUjNRWGRuWjBWTENrRnZTVUpCVVVOWFdVRk9Xa2hGTDB3NWEwMUNjbGRoVTNCT1FXWmpSRmxGUVVkaWMwaE5TRXBNTVROVmF6RklNRVJ4VXl0UVdVRlBTSEZuU1ZSbVVHTUtObXBYZUVoT1ExbHdiWEpZZDFwamFGRkRWVmx2Y1ZGRWJVaFVRamwwUjBnemIyZFNPRTlOYWtOa05VaDJjRkJUZVRab1ZHSnpXR3hEVEdabU5EVndlUXBQVGxWVEwxcGtkMmRaUTNkdk9HaHpjVzF4U2xGbFdHczJWRUozVFdkSGNUVkNjbmRvTldscmMweFhibGx6YlZGbVVHSlhOR2NyVmpoamNtVTJhVUl3Q2t0blp6UkhSMU5LY2pOWmRWWlNhVWt3Y1dnMFMyUXpURXR1TUVOTFVHOUtiakZsZVRadFdFOVNURXRrYlV0R2FDOTVTMFYzTDFGNVNFMXJlamR0VEdzS1FrbFVVbW93U0VoTGFFNDBUVGhxUVU0clNucFhURTFuTVRsbVpuUk5WVk5pUjNBNVdIaHhibTVWTkVKNFZ6TnRiMWMyVGtORmRHeHJiR1pFTDNOMVNRcEVZMmhzZUhGWVlraDNkbFJaZDFSMGRIZHVWME5UWmxOeFlVTjJRV2ROUWtGQlIycFhWRUpZVFVFMFIwRXhWV1JFZDBWQ0wzZFJSVUYzU1VOd1JFRlFDa0puVGxaSVVrMUNRV1k0UlVKVVFVUkJVVWd2VFVJd1IwRXhWV1JFWjFGWFFrSlVWbkJHZURkc1EyUTFhRmxxWlV4eVNEbE1SMkU0UzFKYVQyRkVRVllLUW1kT1ZraFNSVVZFYWtGTloyZHdjbVJYU214amJUVnNaRWRXZWsxQk1FZERVM0ZIVTBsaU0wUlJSVUpEZDFWQlFUUkpRa0ZSUTAxSVRGbFlaV3RwT0FwalZqQkphM1YzYVVKU1NubGthRlpwU25OaVlUSmFNV1kyWkUxb2NGazJSMFYwY21VNFlsQTNTR1ExU1dOaFZEWlBUekJoVERWQk1tVk1jVmxLVW5WRkNtdDJZM3A2WVZWRFkweDRXREk0WlcxWVdqaE9UM2gyZWpJdmRFTnNTalpHUW14SGFXdHJMMVZ5ZVhacWJrdGxUMUZCV1ZGVFJUUlZjbGhJVkhwRGIwUUtTMk5TY1U5MVpIbFpjRGRTYTB4QmJWTXhiakp3Y0hCVGNXMXljelJvY2twTU5YUnZWemRNVDJ4elNWYzBWak1yVFhKaFVucHllVVZqVmxKTFMwUklRZ3BNUTFOaFNFRnFVV05hVm05c1JtTlRVRWRaWVdZNVlYRm5hSEUzZDFCVFNqQjJMek5DTHpGaWNUSXlhVVZoVkRjMk9ITXpNR3hRVTJseFUyVXJOVmxTQ210WE5HTTFibnBCVTJ0SFNVbHVVVWx1YzJwcGEzRmlkQ3RDSzFoVGVYSnJZMmR2TWxKVWRrSkJVMUo1V0V0S1dFOXNOR1p4V1N0blQzTlBVRm93VldjS2RFVkJRM2xTSzFJNWQyTklDaTB0TFMwdFJVNUVJRU5GVWxSSlJrbERRVlJGTFMwdExTMEsKICAgIHNlcnZlcjogaHR0cHM6Ly8xMC4wLjcuMTAwOjY0NDMKICBuYW1lOiBrdWJlcm5ldGVzCmNvbnRleHRzOgotIGNvbnRleHQ6CiAgICBjbHVzdGVyOiBrdWJlcm5ldGVzCiAgICB1c2VyOiBrdWJlcm5ldGVzLWFkbWluCiAgbmFtZToga3ViZXJuZXRlcy1hZG1pbkBrdWJlcm5ldGVzCmN1cnJlbnQtY29udGV4dDoga3ViZXJuZXRlcy1hZG1pbkBrdWJlcm5ldGVzCmtpbmQ6IENvbmZpZwpwcmVmZXJlbmNlczoge30KdXNlcnM6Ci0gbmFtZToga3ViZXJuZXRlcy1hZG1pbgogIHVzZXI6CiAgICBjbGllbnQtY2VydGlmaWNhdGUtZGF0YTogTFMwdExTMUNSVWRKVGlCRFJWSlVTVVpKUTBGVVJTMHRMUzB0Q2sxSlNVUkpWRU5EUVdkdFowRjNTVUpCWjBsSlNsRXZSSGRGVDBGVFIzZDNSRkZaU2t0dldrbG9kbU5PUVZGRlRFSlJRWGRHVkVWVVRVSkZSMEV4VlVVS1FYaE5TMkV6Vm1sYVdFcDFXbGhTYkdONlFXVkdkekI1VFhwRmVFMUVVWGxOUkUweFRsUk9ZVVozTUhsT1JFVjRUVVJOZVUxRVVYZE9WRnBoVFVSUmVBcEdla0ZXUW1kT1ZrSkJiMVJFYms0MVl6TlNiR0pVY0hSWldFNHdXbGhLZWsxU2EzZEdkMWxFVmxGUlJFVjRRbkprVjBwc1kyMDFiR1JIVm5wTVYwWnJDbUpYYkhWTlNVbENTV3BCVGtKbmEzRm9hMmxIT1hjd1FrRlJSVVpCUVU5RFFWRTRRVTFKU1VKRFowdERRVkZGUVhWbVVYaHhPSEZIU2pOT2FGb3JhMElLTmxSVFFYaE9aRnBZVHpoTVVEWnhaazk2UWtkUlRYUm5jWFpTYjI5QmIzcEhVVTU1VlRkUWRTOTRTWEJUT0RWUVZVYzFOSE40V1ZCMFREaE5hMU5NTHdvNGFFazVhV3hyZWtSVmVraDZVazV2UjJVelJVSlFSemcwUTJST1JUZHJSVXMxUTNJek56bFdka1U0VVVoeFNrSllOemhKTmpsYU1XaFNXVE15S3pGakNqbGFSM2t4WVRocE1GZG1VMncxTVVKUFR6QlRiWGx4YVZWVGVIaFJaRXcxYmxac1JuUnhNelZsTmtkamRYSmFPRXA0Yld0d1kwODJXVFpGZG5aaFpXWUtPR2RwZGxoelRGSXhSR3RMUkdJemQxSnNjVzVzZVdGTFIwZHlSbGRVVURZclVuWmtOMkZSVkVkdmJrbG9jMUZLTjNOak9UZEVaREZ4YjFobGFGRkZNZ3B0ZW5selZXcFVURFpzWkhScWFYTnhaMGR5V25SSllYSnFaMEp0Wm1KaU5XWmxORWt3VmtWWVlXVlFObXh5Vnpaa2IzRjZkemN3ZVROd09ERnFlbVZUQ21KRGNVNW9kMGxFUVZGQlFtOHhXWGRXUkVGUFFtZE9Wa2hST0VKQlpqaEZRa0ZOUTBKaFFYZEZkMWxFVmxJd2JFSkJkM2REWjFsSlMzZFpRa0pSVlVnS1FYZEpkMFJCV1VSV1VqQlVRVkZJTDBKQlNYZEJSRUZtUW1kT1ZraFRUVVZIUkVGWFowSlVWbkJHZURkc1EyUTFhRmxxWlV4eVNEbE1SMkU0UzFKYVR3cGhSRUZPUW1kcmNXaHJhVWM1ZHpCQ1FWRnpSa0ZCVDBOQlVVVkJhMVIwTW5Wd1MxZzFUbnBOZUUwMlowbDRRV2hYTlhGbmNtRXpTRE50WjFCTFkwaFpDalIxUW1SaFZEZEVibkY2VUVsS1psUnNTemREVW1wb05rdFZUV296YzJSb2MyRmpObE4zZHpjMGFHWkhOazVyY2pSWVpIRk9TMWsxVUd4MFNraEpUMUVLWkZKVWJ6VjZXRXhsYlV0NmVGcFNRMnBqZVhwUFpWQTNXbEpuWVZBdmNXMVVhMWhQZEZOWlRsUmFRa05QVFhKRGMxWTVVa0ZQVGtKVlVUZDNjR0ZTTWdvMVMxRnlaM2gxYzJ4bU55OTRhV3R1TldGelluVkRjVzlRV2s4ME1WbzJWSEpaYUc5SlIzSmpSQ3Q0TmtoUFYwNHhkbmRzWmxkd2VEVXlNVWRrY25ORENtUnZaREY1U0ZCTVVVOHdXVTlOTVVOMmQzTXZaMjB6ZGswelJtcHJkMDh5UmxCSk1UQlVlRUV2TmpnNVUzTlJPRVpTVVhGMk5VRjFOMVZIVTBwclJFVUtTRFpGU0ZGWVVGaFVjM1ZWUTBOR1R6WlJkbmN3VDNJclVXNHhhRUpTY0RKb1VXUm5PVkZxTHk5RlFsZFFVbU0wT1hjOVBRb3RMUzB0TFVWT1JDQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENnPT0KICAgIGNsaWVudC1rZXktZGF0YTogTFMwdExTMUNSVWRKVGlCU1UwRWdVRkpKVmtGVVJTQkxSVmt0TFMwdExRcE5TVWxGYjNkSlFrRkJTME5CVVVWQmRXWlJlSEU0Y1VkS00wNW9XaXRyUWpaVVUwRjRUbVJhV0U4NFRGQTJjV1pQZWtKSFVVMTBaM0YyVW05dlFXOTZDa2RSVG5sVk4xQjFMM2hKY0ZNNE5WQlZSelUwYzNoWlVIUk1PRTFyVTB3dk9HaEpPV2xzYTNwRVZYcEllbEpPYjBkbE0wVkNVRWM0TkVOa1RrVTNhMFVLU3pWRGNqTTNPVloyUlRoUlNIRktRbGczT0VrMk9Wb3hhRkpaTXpJck1XTTVXa2Q1TVdFNGFUQlhabE5zTlRGQ1QwOHdVMjE1Y1dsVlUzaDRVV1JNTlFwdVZteEdkSEV6TldVMlIyTjFjbG80U25odGEzQmpUelpaTmtWMmRtRmxaamhuYVhaWWMweFNNVVJyUzBSaU0zZFNiSEZ1YkhsaFMwZEhja1pYVkZBMkNpdFNkbVEzWVZGVVIyOXVTV2h6VVVvM2MyTTVOMFJrTVhGdldHVm9VVVV5YlhwNWMxVnFWRXcyYkdSMGFtbHpjV2RIY2xwMFNXRnlhbWRDYldaaVlqVUtabVUwU1RCV1JWaGhaVkEyYkhKWE5tUnZjWHAzTnpCNU0zQTRNV3A2WlZOaVEzRk9hSGRKUkVGUlFVSkJiMGxDUVZGRE1Vd3JVSEZKVDJKQmNpdE1PQXBNV1ZOSlNqTnNSVEI0VVcxM2JVTjVaekY1ZEdadFkyeHZWVlZ4Y21kVU16QTFhR2RUWjFKMU0wbGpSMFZFYzJWMWQwbzNUek5xZEM5d1VVWkxTbHBzQ2tsTVdrVjBNVTE0V2paTFpqRmxNV3QyUVVWWFMzQTVlSFJrTWpCb00ySktkbVF2TjFaMllYaHBRM2hSTVV4VEwwNUtaMEZhUWt0alVWSlhNMUJMVUcwS04yZDRRekYwYkhFM2EwRTVla042WkZGQlQyRktVakphUWxaNWRFODNhR0Z4UVdWTUwwOVdiU3R5WTBJeFNrWjNRbTFHUkd4VVF6RTFUM2xzTjBkR1NBcDZlV1p6ZDBwWlZXVTFha0paWlRGRGVWVlNZMjFxUkdzeVYwUXZORmN3WWl0d1VGZFRhM1ZpTlVSRE1FMWpPVGRtYzJKSVVqZElLMjgxVDJSdlZuSnFDazAzVVVveE9FaG1kbTUzU2pKVWRpdHlkVWhGWldoQ1ZrMVRXQzgwVFc4NVluUkJUMHByY1cxQ2JUaFZSelZ0VVRKeGJHTTJkSFYxYkV4bk9HeEVaazhLUlhoUE1VRllla3BCYjBkQ1FVNXlVM0Y1ZVVFelpXcENZV0owTkdsc1lsQnBTVFpaV0U1RVZpOUhPWE40YjJOaGRDOXJhWHBsTW14eGJtZGtNMDFSU1FwaVZteGFPVTQxV0c5VVpXUlZUakJCV1dkcU1Xc3Jla1V5YUVvdlNUZHNhbE5xU1RoSVlVOHpZa2hLUTBNMWRVNDFSRGhvY1hSakswMTVOV2NyY21SSUNtNXpjSGRpVFZRd05FTkxNWEEyVlZjeVFXcGhjR01yUlhSUVRHODJhRlJXV0hjdk0wMXNhM2hhV1ZSR1dFcDFWU3RwYVdveVlVMVdRVzlIUWtGT2JVd0tPRlJtY2k5bFUzQmhjV1l6WkZkblNWVXdVVzUxT0M5aWVFVnBkMFV5ZVRnMVdIQmFkVFpKYWpSNFVWWjFVV3BDZWtRM1NXTXpiVWh3UVRSNlFYRnlkZ3B3V0dRM1FraHJRbXRRVkdaRGVrOVJSRGhEWmxGMVNUazJWRFZ6VlRSb2VFOXJXakZEVlROMVNEZ3lMMDlMTkdSSFlXYzBOVzk1WVdWVWJEY3dhVUpQQ2taRmIweDBZMk5GZUdWc1dXTkdialZJUkdvMVRsZExaM1ppV2pRMFZXUkxjRTQyUnpaalZYSkJiMGRCUkdaRGFuaFFZMFZ5UVVrclUxSnZXblpuVVVVS2EwRnFNbmxNYzBwR1RsRmhTbHB6TTBsNVJISmxNM014VXl0bGFrYzNNM0J6VjNGQmNHWldlRXROYWxZd1R6VnZWVWRtVFc0MlZ5dDFjbFJ6ZDNFcmJncHNNa2R5UW1KWU1uRTJVek5rY0ZwdlpIWnJaa2RCTUROV00yOUpTSE55Y1U4M01VUjNTbmhWT1hkRldtZ3ZVRmh0VjBSTlpGWmlaamREVjFsWEwwWnVDakVyTWtsblNsRmpRVzlTVFV4UlJXeHRjR04wYTNrd1EyZFpRVWxXY1ZSWlZDdE9ZbU5IVERKTVZGbERNWE0zWVZCbGRXc3JMMlEyUldOWGN6RldReklLWWpsdlVsUlBOMWhTYWpOb1lVRjNPRWM1VEZKU1lrY3dSMkZDZDJwT056WjRWM2hYWkhkcWJsZGxWa1ZDVEVkV05FbFVabmg0S3pWc1RURmxNMWhuZFFwWGVUUlJTSEZDTldOdGNuQjNXbEJMVEhWUmJsZzBVbXd2Tm1vclRTOHZla1p3TDFKRlVUTkZPQ3MzWVdoQlNHYzNVM2gxY1RGeFlVMHJablZ2TlROUENrdGlNbk5KVVV0Q1owZEZVa2ROV2xkNVZUTXdSVmxFTmxVelYxVjJlVEZSWWxSU1kyWTBSR3hvTjFkWlVWWlZablk0ZFdoRVNuRTFWRWxsUkVSaWNrb0tLMVp1ZVhFeFoxQXJSWE5hVXpSaVpVdE5VVmxCV2xsQlJuVjFjamhIV2psQlVsZ3pkbnBWYzBSU05XOUZjMWRtZVZvekx5dGxlVkJaV0VReGNXUmhSd293YkdodFQxbzFRakZRVjIxRFFraFBaVEYzTldGVlVGaDVXRVpCZUVSQ2MyVjRhM2Q0VlhaNFMxaFdTRFpWYkVaS1JWVlBDaTB0TFMwdFJVNUVJRkpUUVNCUVVrbFdRVlJGSUV0RldTMHRMUzB0Q2c9PQo=

```



```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dockerconfig-secret
type: Opaque
data:
  config.json: ewoJImF1dGhzIjogewoJCSJyZWdpc3RyeS5jbi1oYW5nemhvdS5hbGl5dW5jcy5jb20iOiB7CgkJCSJhdXRoIjogIk1Ua3pORGcwTkRBME5FQnhjUzVqYjIwNmJYVnJaVFkyTmpZPSIKCQl9Cgl9Cn0=

```



```yaml
apiVersion: v1  
kind: PersistentVolumeClaim  
metadata:  
  name: sonar-cache-pvc  
spec:  
  accessModes:  
    - ReadWriteMany
  resources:  
    requests:  
      storage: 1Gi 
  storageClassName: nfs-client 

```



```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: git
    image: registry.cn-hangzhou.aliyuncs.com/tool-bucket/tool:git
    command:
    - cat
    tty: true              
  - name: golang
    image: golang:1.16.5
    command:
    - cat
    tty: true  
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ["/busybox/sh"]
    tty: true
    volumeMounts:  
    - mountPath: "/kaniko/.docker"  
      name: "dockerconfig-volume"                                      
  - name: sonar
    image: sonarsource/sonar-scanner-cli
    command:
    - sleep
    args:
    - 99d 
    volumeMounts:
    - mountPath: "/opt/sonar-scanner/.sonar/cache"
      name: "volume-0"   
  - name: kubectl  
    image: registry.cn-hangzhou.aliyuncs.com/tool-bucket/tool:kubectl
    command:  
    - cat  
    tty: true  
    volumeMounts:  
    - mountPath: "/root/.kube"  
      name: "kubeconfig-volume"                                  
  volumes:
  - name: "volume-0"
    persistentVolumeClaim:
      claimName: "sonar-cache-pvc"   
  - name: "kubeconfig-volume"  
    secret:  
      secretName: "kubeconfig-secret"    
  - name: "dockerconfig-volume"  
    secret:  
      secretName: "dockerconfig-secret"   
```

```yaml
#!groovy

@Library('jenkins-shared-library@main') _

def tools = new org.color()
def checkout = new org.checkout()

pipeline {

  agent {
    kubernetes {
      retries 2
      yamlFile 'KubernetesPod.yaml'   
    }
  }

  parameters {
    choice choices: ['main', 'pre', 'test'], name: 'branch_name'
  }

  options {
    timestamps()      
    parallelsAlwaysFailFast()
    timeout(time: 600, unit: 'SECONDS') 
    disableConcurrentBuilds(abortPrevious: true) 
    buildDiscarder(logRotator(numToKeepStr: '30'))
    skipDefaultCheckout() 
  }

  environment {
    String year = new Date().format("yyyy") 
    String month = new Date().format("MMdd") 
    String day = new Date().format("HHmm") 
    String second = new Date().format("ss")		    
    giturl = "http://10.0.7.30/golang/go.git" 
    images_head = "registry.cn-hangzhou.aliyuncs.com/tool-bucket/tool"   
    ImageTag = "${images_head}:${BUILD_USER}-${branch_name}-${year}${month}${day}${second}-${BUILD_TAG}"           
  }

  post {
    failure {
      script {
        manager.addShortText("${BUILD_NUMBER}次构建! 构建失败!!!") 
      }
    }
    success {
      script {
        manager.addShortText("${BUILD_NUMBER}次构建! 构建成功!!!")
      }               
    }                                  
    aborted {
      script {
        manager.addShortText("${BUILD_NUMBER}次构建!构建取消!!!")
      }
    }
    always {
      script {
        manager.addShortText("构建用户: ${BUILD_USER}")
      }
    }	        
  } 

  stages {
    stage('1.克隆代码') {
      steps {
        container('git') {
          script {
            tools.PrintMessage("1.克隆代码","blue")   
            checkout.scm(branch_name,giturl)   
          }
        }
      }
    }        
    stage('2.SonarQube扫描') {
      steps {
        container('sonar') {
          script {
            tools.PrintMessage("2.SonarQube扫描","blue") 
            withSonarQubeEnv('sonarqube') {
              sh '''
                sonar-scanner \
                -Dsonar.projectKey=test-go \
                -Dsonar.projectName=test-go \
                -Dsonar.projectVersion=test-go-${BUILD_NUMBER} \
                -Dsonar.ws.timeout=30 \
                -Dsonar.sources=. \
                -Dsonar.sourceEncoding=UTF-8
                sleep 3
              '''
            }            
          }
        }
      }
    }    
    stage("3.质量阀控制"){
      steps {
        container('sonar') {
          script {      
            tools.PrintMessage("3.质量阀控制","blue")
            timeout(time: 10, unit: 'SECONDS') { 
              def qg = waitForQualityGate('sonarqube') 
              if (qg.status != 'OK') {
                error "未通过Sonarqube的代码质量阈检查，请及时修改！failure: ${qg.status}"
              }
            }
          }
        }
      }
    }    
    stage('4、构建镜像') {
      steps {
        container('kaniko') {
          script {
            tools.PrintMessage("4.构建镜像","blue")
            sh """
              /kaniko/executor --dockerfile=Dockerfile \
              --destination=${ImageTag} \
              --context=. \
              --cache-copy-layers \
              --cache=true \
              --cache-repo=${images_head}
            """
          }
        }
      }
    }       
    stage('5.服务部署') {  
      steps {  
        container('kubectl') {  
          script {
            tools.PrintMessage("5.服务部署","blue")
            sh """
              sed -i "s#image: .*#image: ${ImageTag}#" deploy.yaml
              kubectl apply -f deploy.yaml 
              # kubectl create secret generic kubeconfig-secret --from-file=config=/root/.kube/config 
            """            
          }
        }  
      }  
    }     
    stage('6.MeterShpere接口测试') {
      steps {
        script {
          tools.PrintMessage("6.MeterShpere接口测试","blue")
          meterSphere method: 'testPlan',
          mode: 'serial', 
          msAccessKey: 'OYPWJwNk9vi6JNiF', 
          msEndpoint: 'http://10.0.7.27:8081/', 
          msSecretKey: '7D0UlTZXXTZMt0fD', 
          openMode: 'auth', 
          projectId: 'a6ad182f-f6f1-44de-bdc7-3e6aef7c9e84', 
          projectName: '', 
          projectType: 'projectId', 
          resourcePoolId: '237d98d4-40b9-11ee-a07c-0242ac1e0a09', 
          testCaseId: '', testCaseName: '', 
          testPlanId: '99355526-981f-4927-bd6d-b37e54eb00c1', 
          testPlanName: '', 
          workspaceId: '4d144486-dc2a-4282-acd6-9b449869eeae'                    
        } 
      }  
    }   
  }   
}

```

