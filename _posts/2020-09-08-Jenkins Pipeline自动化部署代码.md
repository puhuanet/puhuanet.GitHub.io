# Jenkins Pipeline自动化部署代码

```
pipeline {
    /*
    系统：git
    插件：Gitlab Plugin, Subversion Plug-in，Email Extension
    配置：svn凭据, git凭据，ssh凭据, Extended E-mail Notification
    TODO: 处理git空目录
    参考：
        https://blog.csdn.net/weixin_33127753/article/details/88870257
        https://github.com/github/gitignore
        https://plugins.jenkins.io/email-ext/
        https://github.com/jenkinsci/ssh-steps-plugin

    */

    agent any
    
    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        _DEBUG = true
        
        SVN_REPO_DIR = "${env.WORKSPACE}/SVN_REPO"
        SVN_CREDENTIALS_ID = 'svn_JK-test'
        SVN_REPOSITORY_URL = 'http://svn.puhua.net/svn/JK-test'
        
        GIT_REPO_DIR = "${env.WORKSPACE}/GIT_REPO"
        GIT_CREDENTIALS_ID = 'git_JK-test'
        GIT_REPOSITORY_URL = 'http://gitlab.puhua.net/git/jk-test.git'
        GIT_BRANCHE = 'dev'

        APP_CREDENTIALS_ID = 'ssh_JK-test'
        APP_GIT_REPO_DIR = '/opt/GIT_REPO'
        APP_NAME_1 = 'APP1'
        APP_HOST_1 = ''
        APP_PORT_1 = '22'
        APP_NAME_2 = 'APP2'
        APP_HOST_2 = ''
        APP_PORT_2 = ''


    }

    parameters {
        string name: 'JOBNAME', defaultValue: '', description: '任务名称：\n请在上方的文本框中输入“任务名称”\n\n', trim: true
        text name: 'ADD_LIST', defaultValue: '', description: '增加文件、目录：\n请在上方的文本框中输入“需增加的文件、目录”清单\n\n'
        text name: 'RM_LIST', defaultValue: '', description: '移除文件、目录：\n请在上方的文本框中输入“需移除的文件、目录”清单\n\n'
    }

    stages {
        stage("检查参数") {
            when { expression { params.JOBNAME == '' || (params.ADD_LIST == '' && params.RM_LIST == '') } }
            steps {
                script {
                    currentBuild.result = 'ABORTED'
                    error('参数不能为空，请输入任务名称和部署清单。')
                }
            }
        }

        stage("检出") {
            parallel {
                stage("SVN 检出") {
                    steps {
                        script { currentBuild.result = 'SUCCESS' }
                        dir (env.SVN_REPO_DIR) {
                            checkout([
                                $class: 'SubversionSCM', 
                                additionalCredentials: [], 
                                excludedCommitMessages: '', 
                                excludedRegions: '', 
                                excludedRevprop: '', 
                                excludedUsers: '', 
                                filterChangelog: false, 
                                ignoreDirPropChanges: false, 
                                includedRegions: '', 
                                locations: [[
                                    cancelProcessOnExternalsFail: true, 
                                    credentialsId: env.SVN_CREDENTIALS_ID, 
                                    depthOption: 'infinity', 
                                    ignoreExternalsOption: true, 
                                    local: '.', 
                                    remote: env.SVN_REPOSITORY_URL
                                ]], 
                                quietOperation: true, 
                                workspaceUpdater: [$class: 'UpdateUpdater']
                            ])
                        }
                    }
                    post {
                        success { show_info "SVN 检出成功" }
                        unsuccessful { set_unsuccessful(); show_warn "SVN 检出失败" }
                    }
                }

                stage("GIT 检出") {
                    steps {
                        dir (env.GIT_REPO_DIR) {
                            checkout(
                                scm: [$class: 'GitSCM', 
                                    branches: [[name: "*/${env.GIT_BRANCHE}"]], 
                                    doGenerateSubmoduleConfigurations: false, 
                                    extensions: [[$class: 'LocalBranch', localBranch: env.GIT_BRANCHE]], 
                                    submoduleCfg: [], 
                                    userRemoteConfigs: [[
                                        credentialsId: env.GIT_CREDENTIALS_ID, 
                                        url: env.GIT_REPOSITORY_URL
                                    ]]
                                ],
                                changelog: false, 
                                poll: false)
                        }
                    }
                    post {
                        success { show_info "GIT 检出成功" }
                        unsuccessful { set_unsuccessful(); show_warn "GIT 检出失败" }
                    }
                }
            }
        }

        stage ("预处理"){
            parallel {
                stage("预处理部署清单") {
                    steps {
                        script {
                            df_list = []   // 删除文件的清单
                            dd_list = []   // 删除目录的清单
                            ad_list = []   // 新增目录的清单
                            af_list = []   // 新增文件的清单
                            ne_list = []   // 路径不存在的清单

                            // 
                            src_list_add = params.ADD_LIST.split('\n')
                            src_list_rm = params.RM_LIST.split('\n')
                            getDeployList()
                        }
                    }
                    post {
                        success { show_info "预处理部署清单成功" }
                        unsuccessful { set_unsuccessful(); show_warn "预处理部署清单失败" }
                    }            
                }

                stage("GIT 忽略文件") {
                    steps {
                        // dir (env.GIT_REPO_DIR) {
                        script {
                            def gi = "${env.GIT_REPO_DIR}/.gitignore"
                            def fc = """\
# 忽略指定文件

# 忽略指定格式的文件
*.ini

# 忽略指定文件夹
.svn/
.git/
"""
                            writeFile file: gi, text: fc, encoding: "UTF-8"
                        }
                    }
                    post {
                        success { show_info "GIT 忽略文件成功" }
                        unsuccessful { set_unsuccessful(); show_warn "GIT 忽略文件失败" }
                    }
                }
            }
        }


        stage("部署到GIT本地暂存区") {
            steps {
                doDeployStage()
            }
            post {
                success { show_info "部署到GIT本地暂存区成功" }
                unsuccessful { set_unsuccessful(); show_warn "部署到GIT本地暂存区失败" }
            }
        }

        stage("提交/推送到GIT仓库") {
            steps {
                dir (env.GIT_REPO_DIR) {
                    script {
                        withCredentials([usernamePassword(credentialsId: env.GIT_CREDENTIALS_ID, passwordVariable: 'GIT_P', usernameVariable: 'GIT_U')]) {
                            def GIT_P_EC = URLEncoder.encode(GIT_P, "UTF-8")
                            def cmd = """
                                set +x
                                git config --local credential.username ${GIT_U}
                                git config --local credential.helper store "!echo password=${GIT_P_EC}; echo"
                                git config user.name "${GIT_U}"
                                git config user.email "${GIT_U}@puhua.net"
                                git config push.default simple
                                set -x                                
                            """
                            sh label: 'GIT 配置', returnStdout: true, script: cmd
                        }
                        sh label: 'GIT Status', returnStdout: true, script: "git status"
                        sh label: 'GIT 提交到本地仓库', returnStdout: true, script: "git commit -m '${params.JOBNAME} ${env.BUILD_DISPLAY_NAME}'"
                        sh label: 'GIT 推送到远程仓库', returnStdout: true, script: "git push origin ${env.GIT_BRANCHE}:${env.GIT_BRANCHE}"
                    }
                }
            }
            post {
                success { show_info "提交/推送到GIT仓库成功" }
                unsuccessful { set_unsuccessful(); show_warn "提交/推送到GIT仓库失败" }
            }
        }

        stage("部署到APP服务器") {
            parallel {
                stage("APP1") {
                    steps {
                        doDeployApp(env.APP_NAME_1, env.APP_HOST_1, env.APP_PORT_1)
                    }
                    post {
                        success { show_info "部署到APP服务器: APP1成功" }
                        unsuccessful { set_unsuccessful(); show_warn "部署到APP服务器: APP1失败" }
                    }
                }

                stage("APP2") {
                    steps {
                        show_info "部署到APP服务器: APP2"
                        // doDeployApp(env.APP_NAME_2, env.APP_HOST_2, env.APP_PORT_2)
                    }
                    post {
                        success { show_info "部署到APP服务器: APP2成功" }
                        unsuccessful { set_unsuccessful(); show_warn "部署到APP服务器: APP2失败" }
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                // env.getEnvironment().each { name, value -> println "Name: $name -> Value $value"}
                df_str = ''
                dd_str = ''
                ad_str = ''
                af_str = ''
                ne_str = ''

                try {
                    df_str = df_list.join('\n')
                    dd_str = dd_list.join('\n')
                    ad_str = ad_list.join('\n')
                    af_str = af_list.join('\n')
                    ne_str = ne_list.join('\n')
                } catch(e) { }
                    
                try {
                    subject = "${params.JOBNAME} ${env.BUILD_DISPLAY_NAME} ${currentBuild.result}"
                    body = """
                        <!DOCTYPE html>
                        <html>
                        <head>
                            <meta charset="UTF-8">
                            <title>邮件通知</title>
                            <style>
                            h4 {
                                font-size: 16px;
                                color: #00007f;
                                text-shadow: 1px 0px 1px #e6e6e6;
                            }

                            ul {
                                list-style-type: square;
                                padding-left: 20px;
                                margin: 0px;
                            }

                            ul li {
                                padding-bottom: 5px;
                            }

                            #report {
                                font-size: 14px;
                                width: 90%;
                                background-color: antiquewhite;
                                margin: auto;
                                padding: 25px;
                                border-radius: 25px;
                                box-shadow: 1px 1px 1px #e6e6e6;
                            }

                            .log {
                                font-size: 12px;
                                margin-left: 20px;
                                margin-bottom: 20px;
                            }
                            </style>
                        </head>
                        <body>
                            <div width="95%">
                            <div>
                                <p> <strong>各位主管,</strong></p>
                                <pre>    以下为自动化部署代码信息：</pre>
                            </div>
                            </div>
                            <div id="report">
                            <div>
                                <h4>
                                部署结果：<span style="color: black; background-color: white;padding: 2px;">${currentBuild.result}</span>
                                <hr size="1" style="width: 100%;" />
                            </div>
                            <div>
                                <h4> 部署信息 </h4>
                                <ul>
                                <li>任务名称：${env.JOBNAME}</li>
                                <li>部署编号：${env.BUILD_DISPLAY_NAME}</li>
                                <li>部署分支：${env.GIT_BRANCHE}</li>
                                <li>部署耗时：${currentBuild.durationString}</li>
                                <li>部署 Url：<a href="${env.RUN_DISPLAY_URL}">${env.RUN_DISPLAY_URL}</a></li>
                                </ul>
                            </div>
                            <div>
                                <h4> 部署清单 </h4>
                                <div class="log"><pre>${params.ADD_LIST}</pre></div>
                            </div>
                            <div>
                                <h4> 预处理部署清单 </h4>
                                <ul>
                                <li>删除文件：</li>
                                <div class="log"><pre>${df_str}</pre></div>
                                <li>删除目录：</li>
                                <div class="log"><pre>${dd_str}</pre></div>
                                <li>新增目录：</li>
                                <div class="log"><pre>${ad_str}</pre></div>
                                <li>新增文件：</li>
                                <div class="log"><pre>${af_str}</pre></div>
                                <li>路径不存在：</li>
                                <div class="log"><pre>${ne_str}</pre></div>
                                </ul>
                            </div>
                            <div>
                                <h4> 任务日志 </h4>
                                <div class="log"> 请查阅邮件附档：build.log</div>
                            </div>
                            </div>
                        </body>
                        </html>
                    """
                    emailext( 
                        mimeType: 'text/HTML',
                        attachLog: true, 
                        to: 'bcc:jacky@puhua.net',
                        recipientProviders: [requestor(),culprits(),developers()], 
                        subject: subject, 
                        body: body
                    )
                    // cleanWs notFailBuild: true
                } catch(e) {show_warn e}
            }
        }
    }
}

/* 显示debug */
def show_debug(msg) {
    if (env._DEBUG) { println "[DEBUG] ${msg}" }
}

/* 显示信息 */
def show_info(msg) {
    println "[INFO] ${msg}"
}

/* 显示警告 */
def show_warn(msg) {
    println "[WARN] ${msg}"
}

/* 阶段不成功 */
def set_unsuccessful() {
    currentBuild.result = 'FAILURE'
}

/* 预处理的部署清单文件 */
def getDeployList() {
    getDeployList_ADD()
    getDeployList_RM()
}

def getDeployList_ADD() {
    // 忽略";"开头的注释行和空行
    def pattern = ~/^(?!;)\s*(.+)$/
    src_list_add.each {
        def line = it.toString()
        def match = line =~ pattern
        if (match) {
            def path = match.group(1)
            File file = new File(env.SVN_REPO_DIR + path)
            show_debug env.SVN_REPO_DIR + path
            if (file.exists()) { 
                if (file.isFile()) { af_list.add(path) } else { ad_list.add(path) }
            } else { 
                show_warn "路径不存在: ${path}"
                ne_list.add(path)
            }
        }
    }
}

def getDeployList_RM() {
    // 忽略";"开头的注释行和空行
    def pattern = ~/^(?!;)\s*(.+)$/
    src_list_rm.each {
        def line = it.toString()
        def match = line =~ pattern
        if (match) {
            def path = match.group(1)
            File file = new File(env.GIT_REPO_DIR + path)
            show_debug env.GIT_REPO_DIR + path
            if (file.exists()) { 
                if (file.isFile()) { df_list.add(path) } else { dd_list.add(path) }
            } else { 
                show_warn "路径不存在: ${path}"
                ne_list.add(path)
            }
        }
    }
}

/* 部署到本地暂存区 */
def doDeployStage() {
    // 清除本地暂存区
    dir (env.GIT_REPO_DIR) {
        sh label: 'GIT 清除本地暂存区', returnStdout: true, script: 'git rm -r --cached .'
    }

    // 删除文件
    dir (env.GIT_REPO_DIR) {
        if (!df_list.isEmpty()) {
            df_list.each {
                def df = "${env.GIT_REPO_DIR}${it.toString()}"
                def cmd = "sudo /usr/bin/rm '${df}'"
                sh label: '从GIT工作区删除文件', returnStdout: true, script: cmd
            }
        }
    }

    // 删除目录
    dir (env.GIT_REPO_DIR) {
        if (!dd_list.isEmpty()) {
            dd_list.each {
                def dd = "${env.GIT_REPO_DIR}${it.toString()}"
                def cmd = "sudo /usr/bin/rm -r '${dd}'"
                sh label: '从GIT工作区删除目录', returnStdout: true, script: cmd
            }
        }
    }

    // 新增目录
    dir (env.SVN_REPO_DIR) {
        if (!ad_list.isEmpty()) {
            def dst = env.GIT_REPO_DIR
            ad_list.each {
                def src = ".${it.toString()}"
                def cmd = "sudo /usr/bin/cp -r --parents '${src}' '${dst}'"
                sh label: '复制目录到GIT工作区', returnStdout: true, script: cmd
            }
        }
    }
    // 新增文件
    dir (env.SVN_REPO_DIR) {
        if (!af_list.isEmpty()) {
            def dst = env.GIT_REPO_DIR
            af_list.each {
                def src = ".${it.toString()}"
                def cmd = "sudo /usr/bin/cp --parents '${src}' '${dst}'"
                sh label: '复制文件到GIT工作区', returnStdout: true, script: cmd
            }
        }
    }

    // 更新本地暂存区
    dir (env.GIT_REPO_DIR) {
        sh label: 'GIT 更新本地暂存区', returnStdout: true, script: 'git add --all'
    }
}

/* 部署到APP服务器 */
def doDeployApp(name, host, port) {
    show_info "部署到APP服务器: ${name}"
    def remote = [:]
    remote.name = name
    remote.host = host
    remote.port = port.toInteger()
    remote.allowAnyHosts = true
    retryCount = 2
    encoding = 'UTF-8'
    withCredentials([usernamePassword(credentialsId: env.APP_CREDENTIALS_ID, passwordVariable: 'SSH_P', usernameVariable: 'SSH_U')]) {
        remote.user = SSH_U
        remote.password = SSH_P
        withCredentials([usernamePassword(credentialsId: env.GIT_CREDENTIALS_ID, passwordVariable: 'GIT_P', usernameVariable: 'GIT_U')]) {
            def GIT_P_EC = URLEncoder.encode(GIT_P, "UTF-8")
            def cmd = """
                cd ${env.APP_GIT_REPO_DIR}
                set +x
                git init
                git config --local credential.username ${GIT_U}
                git config --local credential.helper store "!echo password=${GIT_P_EC}; echo"
                git config user.name "${GIT_U}"
                git config user.email "${GIT_U}@puhua.net"
                git config push.default simple
                set -x
                git status
                git pull ${env.GIT_REPOSITORY_URL} ${env.GIT_BRANCHE}:${env.GIT_BRANCHE}
            """
            sshCommand remote: remote, command: cmd
        }
    }
}
```