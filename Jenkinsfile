pipeline {

    agent {
        kubernetes {
            defaultContainer 'maven'
            yamlFile 'agent.yaml'
        }
    }

    environment {
        GITHUB_TOKEN  = 'edc8b6f756406d8231c3b806c97de1684c273b3a'
    }

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '500'))
        timeout(time: 1, unit: 'HOURS')
    }

    parameters {
        string(name: 'VERSION_TO_BE_RELEASED',
               defaultValue: '',
               description: 'the version of presto-docs to be released, such as 0.279'
        )
        booleanParam(name: 'MARK_AS_CURRENT',
                     defaultValue: false,
                     description: 'mark the new version as current'
        )
        string(name: 'VERSION_CURRENTLY_RELEASED',
               defaultValue: '',
               description: 'the version of presto-docs that has been released, such as 0.278'
        )
    }

    stages {
        stage ('Setup') {
            steps {
                sh 'apt update && apt install -y curl git unzip'
            }
        }

        stage ('Load Presto Repo') {
            steps {
                checkout $class: 'GitSCM',
                         branches: [[name: '*/master']],
                         doGenerateSubmoduleConfigurations: false,
                         extensions: [[
                             $class: 'RelativeTargetDirectory',
                             relativeTargetDir: 'presto'
                         ]],
                         submoduleCfg: [],
                         userRemoteConfigs: [[
                             url: 'https://github.com/prestodb/presto.git'
                         ]]
                sh '''
                    cd presto
                    git config --global --add safe.directory ${WORKSPACE}/presto
                '''
            }
        }

        stage ('Load Website Repo') {
            steps {
                checkout $class: 'GitSCM',
                         branches: [[name: '*/source']],
                         doGenerateSubmoduleConfigurations: false,
                         extensions: [[
                             $class: 'RelativeTargetDirectory',
                             relativeTargetDir: 'prestodb.github.io'
                         ]],
                         submoduleCfg: [],
                         userRemoteConfigs: [[
                             url: 'https://github.com/prestodb/prestodb.github.io.git'
                         ]]
                sh '''
                    cd prestodb.github.io
                    git config --global --add safe.directory ${WORKSPACE}/prestodb.github.io
                    git config --global user.name "OSS CI Infra bot"
                    git config --global user.email "wanglinsong@gmail.com"
                    git checkout -b source
                '''
            }
        }

        stage ('Update Docs') {
            steps {
                sh '''#!/bin/bash -ex
                    cd prestodb.github.io
                    ls -al

                    PRESTO_GIT_REPO=../presto
                    TARGET=website/static/docs/${VERSION_TO_BE_RELEASED}
                    CURRENT=website/static/docs/current

                    if [[ -e ${TARGET} ]]; then
                        echo "already exists: ${TARGET}"
                        exit 100
                    fi

                    curl -O https://repo1.maven.org/maven2/com/facebook/presto/presto-docs/${VERSION_TO_BE_RELEASED}/presto-docs-${VERSION_TO_BE_RELEASED}.zip
                    unzip presto-docs-${VERSION_TO_BE_RELEASED}.zip
                    mv html ${TARGET}
                    unlink ${CURRENT}
                    ln -sf ${VERSION_TO_BE_RELEASED} ${CURRENT}
                    git add ${TARGET} ${CURRENT}
                    git status

                    DATE=$(TZ=America/Los_Angeles date '+%B %d, %Y')
                    echo "Update the version number and stats in javascript for rendering across the site"
                    VERSION_JS=website/static/static/js/version.js

                    echo "const presto_latest_presto_version = '${VERSION_TO_BE_RELEASED}';" > ${VERSION_JS}
                    GIT_LOG="git -C ../presto log --use-mailmap ${VERSION_CURRENTLY_RELEASED}..${VERSION_TO_BE_RELEASED}"
                    NUM_COMMITS=$(${GIT_LOG} --format='%aE' | wc -l | awk '{$1=$1;print}')
                    NUM_CONTRIBUTORS=$(${GIT_LOG} --format='%aE' | sort | uniq | wc -l | awk '{$1=$1;print}')
                    NUM_COMMITTERS=$(${GIT_LOG} --format='%cE' | sort | uniq | wc -l | awk '{$1=$1;print}')
                    echo "const presto_latest_num_commits = ${NUM_COMMITS};" >> ${VERSION_JS}
                    echo "const presto_latest_num_contributors = ${NUM_CONTRIBUTORS};" >> ${VERSION_JS}
                    echo "const presto_latest_num_committers = ${NUM_COMMITTERS};" >> ${VERSION_JS}
                    echo "const presto_latest_date = '${DATE}';" >> ${VERSION_JS}
                    cat ${VERSION_JS}
                    git add ${VERSION_JS}
                '''
            }
        }

        stage ('Push Updates') {
            steps {
                sh '''
                    cd prestodb.github.io

                    git status
                    git commit -m "Add ${VERSION_TO_BE_RELEASED} docs"
                    git push -q https://${GITHUB_TOKEN}@github.com/prestodb/prestodb.github.io.git source:source
                '''
            }
        }
    }
}
