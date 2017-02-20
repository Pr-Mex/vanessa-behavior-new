#!groovy
node("slave") {
    def unix = isUnix()

    def encoding = "-encoding=utf-8";
    encoding="";

    stage ("checkout scm") {
        //git url: 'https://github.com/silverbulleters/vanessa-behavior-new.git'
        
        
        checkout scm
        if (env.DISPLAY) {
            println env.DISPLAY;
        } else {
            env.DISPLAY=":1"
        }
        //env.RUNNER_ENV="production";

        if (unix) {sh 'git config --system core.longpaths'} else {bat "git config --system core.longpaths"}

        if (unix) {sh 'git submodule update --init'} else {bat "git submodule update --init"}
    }

    stage ("sonar QA"){

        if (env.QASONAR) {
            println env.QASONAR;
            def sonarcommand = "@\"./../../../tools/hudson.plugins.sonar.SonarRunnerInstallation/Main_Classic/bin/sonar-scanner\""
            withCredentials([[$class: 'StringBinding', credentialsId: env.SonarOAuthCredentianalID, variable: 'SonarOAuth']]) {
                sonarcommand = sonarcommand + " -Dsonar.host.url=http://sonar.silverbulleters.org -Dsonar.login=${env.SonarOAuth}"
            }
            
            // Get version
            def configurationText = readFile encoding: 'UTF-8', file: 'vanessa-behavior/VanessaBehavior/Ext/ObjectModule.bsl'
            def configurationVersion = (configurationText =~ /Версия = "(.*)";/)[0][1]
            sonarcommand = sonarcommand + " -Dsonar.projectVersion=${configurationVersion}"

            def makeAnalyzis = true
            if (env.BRANCH_NAME == "develop") {
                echo 'Analysing develop branch'
            } else if (env.BRANCH_NAME.startsWith("release/")) {
                sonarcommand = sonarcommand + " -Dsonar.branch=${BRANCH_NAME}"
            } else if (env.BRANCH_NAME.startsWith("PR-")) {
                // Report PR issues           
                def PRNumber = env.BRANCH_NAME.tokenize("PR-")[0]
                def gitURLcommand = 'git config --local remote.origin.url'
                def gitURL = ""
                if (unix) {
                    gitURL = sh(returnStdout: true, script: gitURLcommand).trim() 
                } else {
                    gitURL = bat(returnStdout: true, script: gitURLcommand).trim() 
                }
                def repository = gitURL.tokenize("/")[2] + "/" + gitURL.tokenize("/")[3]
                repository = repository.tokenize(".")[0]
                withCredentials([[$class: 'StringBinding', credentialsId: env.GithubOAuthCredentianalID, variable: 'githubOAuth']]) {
                    sonarcommand = sonarcommand + " -Dsonar.analysis.mode=issues -Dsonar.github.pullRequest=${PRNumber} -Dsonar.github.repository=${repository} -Dsonar.github.oauth=${env.githubOAuth}"
                }
            } else {
                makeAnalyzis = false
            }
            if (makeAnalyzis) {
                if (unix) {
                    sh '${sonarcommand}'
                } else {
                    bat "${sonarcommand}"
                }
            }
        } else {
            echo "QA runner not installed"
        }
    }

    def v8version = "--v8version 8.3.10";
    if (env.V8VERSION) {
        v8version = "--v8version ${env.V8VERSION}"
    }
    env.RUNNER_ENV="debug"

    stage ("init"){
        def srcpath = "./lib/CF/83NoSync";
        if (env.SRCPATH){
            srcpath = env.SRCPATH;
        }
        
        def command = "oscript ${encoding} tools/init.os init-dev ${v8version} --src "+srcpath
        timestamps {
            if (unix){
                sh "${command}"
            } else {
                bat "chcp 1251\n${command}"
            }
        }
    }
    
    stage ("compile"){
        //echo "build catalogs"
        command = """oscript ${encoding} tools/runner.os compileepf ${v8version} --ibname /F"./build/ib" ./ ./build/out/ """
        if (unix) {sh "${command}"} else {bat "chcp 1251\n${command}"}       
    }

    def errors = []

    stage ("behavior tests") {
        def testsettings = "VBParams8310UF.json";
        if (env.PATHSETTINGS) {
            testsettings = env.PATHSETTINGS;
        }
        
        // TODO:
        // Придумать, как это сделать красиво и с учетом того, что задано в VBParams837UF.json
        // Стр = Стр + " /Execute " + ПараметрыСборки["EpfДляИнициализацияБазы"] + " /C""InitDataBase;VBParams=" + ПараметрыСборки["ПараметрыДляИнициализацияБазы"] + """";
        def VBParamsPath = pwd().replaceAll("%", "%%") + "/build/out/tools/epf/init.json"
        command = """oscript ${encoding} tools/runner.os run ${v8version} --ibname /F"./build/ib" --execute "./build/out/tools/epf/init.epf" --command "InitDataBase;VBParams=${VBParamsPath}" """
        
        try{
            if (unix){
                sh "${command}"
                
            } else {
                bat "chcp 1251\n${command}"
            }
        } catch (e) {
            errors << "BDD status : ${e}"
        }

        command = """oscript ${encoding} tools/runner.os vanessa ${v8version} --ibname /F"./build/ib" --pathvanessa ./build/out/vanessa-behavior.epf --vanessasettings ./tools/JSON/${testsettings} """
        try{
            if (unix){
                sh "${command}"
            } else {
                env.VANESSA_commandscreenshot='nircmd.exe savescreenshot '
                bat "chcp 1251\n${command}"
            }
        } catch (e) {
            errors << "BDD status : ${e}"
        }
    }
        
    //allure([includeProperties: true, jdk: '', properties: [], reportBuildPolicy: 'ALWAYS', results: [[path: './build/out/allurereport']]])

    
    stage("result"){

        echo "allure"

        command = """allure generate ./build/allurereport -o ./build/htmlpublish"""
        if (unix){ sh "${command}" } else {bat "chcp 1251\n${command}"}
        publishHTML(target:[allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: './build/htmlpublish', reportFiles: 'index.html', reportName: 'Allure report'])
        
        //step([$class: 'CucumberTestResultArchiver', testResults: './build/cucumber/CucumberJson.json'])
        
        echo "artifacts"
        if (errors.size() > 0) {
            currentBuild.result = 'UNSTABLE'
            for (int i = 0; i < errors.size(); i++) {
                echo errors[i]; 
            }
        } else {
            step([$class: 'ArtifactArchiver', artifacts: '**/build/out/*.epf', fingerprint: true])
            step([$class: 'ArtifactArchiver', artifacts: '**/build/out/features/Libraries/**/*.epf', fingerprint: true])
            step([$class: 'ArtifactArchiver', artifacts: '**/build/out/features/Libraries/**/*.feature', fingerprint: true])    
        }
        echo "junit"

        junit '**/build/junitreport/*.xml'
        echo "CucumberTestReportPublisher"
        step([$class: 'CucumberTestReportPublisher', copyHTMLInWorkspace: true, fileExcludePattern: '', fileIncludePattern: '', ignoreUndefinedSteps: false, markAsUnstable: true, reportsDirectory: './build/cucumber/'])
        echo "CucumberReportPublisher"
        step([$class: 'CucumberReportPublisher', classifications: [], failedFeaturesNumber: 0, failedScenariosNumber: 0, failedStepsNumber: 0, fileExcludePattern: '', fileIncludePattern: '**/*.json', jsonReportDirectory: './build/cucumber/', parallelTesting: false, pendingStepsNumber: 0, skippedStepsNumber: 0, trendsLimit: 0, undefinedStepsNumber: 0])

    }

    //junit './build/junitreport/*.xml'
    
    //step($class: 'CucumberTestResultArchiver', testResults: 'glob')
    

    //echo "stable if master, pre-release if have release, nigthbuild if develop"

}