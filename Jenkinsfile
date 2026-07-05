pipeline {
    agent any

    // 定义构建参数
    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: '选择需要构建的 Git 分支')
        // 添加一个布尔型参数，默认不勾选 (false)
        booleanParam(name: 'APPLY_PATCH', defaultValue: false, description: '是否需要应用 APISIX Nacos Lua 补丁并重启网关？(仅当补丁有修改时才需要勾选)')
        text(name: 'CORS_REGEX_PARAM', defaultValue: '', description: '可选。留空则使用仓库 defaults/cors-allow-origins.json；填写时请给合法 JSON 数组（一行即可）。')
        // 与业务/Nacos 所用 Redis 的 requirepass 一致；留空时 setup.sh 仍默认 root（本地兼容）
        password(name: 'REDIS_AUTH_PASSWORD', defaultValue: '', description: 'Redis AUTH 密码（网关 auth.lua 会话查询用）。生产建议改用 withCredentials 注入并留空此项。')
        booleanParam(name: 'DEPLOY_DOCS_ROUTE', defaultValue: false, description: '是否注册文档路由？(仅当开发服务器部署时才需要勾选)')
        booleanParam(name: 'DEPLOY_FRONTEND_ROUTE', defaultValue: false, description: '是否注册前端路由？(仅当不通过对象存储服务部署前端时才需要勾选)')
        string(name: 'FRONTEND_HOST', defaultValue: '', description: '前端服务 IP/Host。DEPLOY_FRONTEND_ROUTE 勾选时必填，留空则跳过注册前端路由。')
        string(name: 'FRONTEND_PORT', defaultValue: '80', description: '前端服务端口。')
    }

    environment {
        CORS_REGEX_JSON = "${params.CORS_REGEX_PARAM}"
    }

    stages {
        stage('1. 拉取代码 (Checkout)') {
            steps {
                echo "开始拉取 ${params.BRANCH_NAME} 分支的部署脚本..."
                checkout scm
            }
        }

        stage('2. 环境检查与准备') {
            steps {
                script {
                    sh '''
                    if ! command -v jq &> /dev/null; then
                        echo "系统缺失 jq，正在下载独立二进制版本..."
                        mkdir -p "$WORKSPACE/bin"
                        # 使用 GitHub 加速下载 jq 的 Linux amd64 版本
                        curl -L -# -o "$WORKSPACE/bin/jq" "https://github.com/jqlang/jq/releases/download/jq-1.8.1/jq-linux-amd64"
                        chmod +x "$WORKSPACE/bin/jq"
                        echo "✅ jq 下载完毕，版本："
                        "$WORKSPACE/bin/jq" --version
                    else
                        echo "✅ jq 工具已就绪。"
                    fi
                    '''
                }
            }
        }

        stage('3. 应用 APISIX 补丁 (Patch)') {
            // 使用 when 指令：只有当构建参数 APPLY_PATCH 被勾选时，才会执行这个阶段
            when {
                expression { return params.APPLY_PATCH == true }
            }
            steps {
                dir('./') {
                    echo "检测到打补丁选项已开启，开始执行 patch.sh ..."
                    sh "chmod +x patch.sh"
                    sh "./patch.sh"
                    echo "✅ 补丁应用完成，网关容器已重启。"
                }
            }
        }

        stage('4. 自动化注册路由 (Setup Routes)') {
            steps {
                dir('./') {
                    echo "开始推送到 APISIX Admin API 注册全局插件与路由..."
                    sh "chmod +x setup.sh"
                    script {
                        if (params.DEPLOY_FRONTEND_ROUTE && !params.FRONTEND_HOST?.trim()) {
                            error('DEPLOY_FRONTEND_ROUTE 已勾选，但 FRONTEND_HOST 为空。请填写前端服务 IP/Host。')
                        }
                    }
                    withEnv([
                        "PATH=${env.WORKSPACE}/bin:${env.PATH}",
                        "REDIS_AUTH_PASSWORD=${params.REDIS_AUTH_PASSWORD}",
                        "DEPLOY_DOCS_ROUTE=${params.DEPLOY_DOCS_ROUTE}",
                        "FRONTEND_HOST=${params.DEPLOY_FRONTEND_ROUTE ? params.FRONTEND_HOST.trim() : ''}",
                        "FRONTEND_PORT=${params.FRONTEND_PORT}",
                    ]) {
                        sh "./setup.sh"
                    }
                    echo "✅ 路由配置同步完成。"
                }
            }
        }
    }

    post {
        success {
            echo "🎉 网关自动化部署流水线执行成功！"
        }
        failure {
            echo "❌ 流水线执行失败，请检查脚本输出日志或确保 APISIX 服务存活！"
        }
    }
}
