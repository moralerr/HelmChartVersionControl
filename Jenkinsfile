@Library('pipeline-commons') _
import pipeline.commons.*

pipeline {
    agent any
    environment {
        GITHUB_TOKEN = credentials('GITHUB_TOKEN')
        REPO_OWNER = 'moralerr'
        REPO_NAME = 'helm-charts'
        CHART_FILE_PATH = 'charts/jenkins/Chart.yaml'.trim()
        BRANCH_NAME = "update-jenkins-helm-chart-${UUID.randomUUID().toString()}"
    }
    triggers {
        cron('H 1 * * *') // Run every day at 1 am
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                        sh """
                        git clone https://${GITHUB_TOKEN}@github.com/${REPO_OWNER}/${REPO_NAME}.git
                        cd ${REPO_NAME}
                        git checkout main
                        """
                    }
                }
            }
        }
        stage('Get Latest Jenkins Helm Chart Version') {
            steps {
                script {
                    latestVersion = utils.getLatestJenkinsHelmChartVersion()
                    echo "Latest Jenkins Helm Chart Version: ${latestVersion}"
                }
            }
        }
        stage('Get Current Helm Chart Info') {
            steps {
                script {
                    currentInfo = utils.getCurrentHelmChartInfo(REPO_OWNER, REPO_NAME, CHART_FILE_PATH, GITHUB_TOKEN)
                    currentVersion = currentInfo.chartVersion
                    dependencyVersion = currentInfo.dependencyVersion
                    echo "Current Helm Chart Version: ${currentVersion}"
                    echo "Current Jenkins Dependency Version: ${dependencyVersion}"
                }
            }
        }
        stage('Update Chart Info') {
            when {
                expression { latestVersion > dependencyVersion }
            }
            steps {
                script {
                    newChartVersion = utils.incrementMinorVersion(currentVersion)
                    // Ensure we are in the correct directory before updating the file
                    dir(REPO_NAME) {
                        utils.updateHelmChartInfo(CHART_FILE_PATH, newChartVersion, latestVersion)
                        sh "git checkout -b ${BRANCH_NAME}"
                        sh "git add ${CHART_FILE_PATH}"
                        sh "git commit -m 'Update Jenkins Helm chart to version ${latestVersion} and increment chart version to ${newChartVersion}'"
                        sh "git push origin ${BRANCH_NAME}"
                    }
                }
            }
        }
        stage('Create Pull Request') {
            when {
                expression { latestVersion > dependencyVersion }
            }
            steps {
                script {
                    utils.createPullRequest([
                        apiUrl: "https://api.github.com",
                        owner: REPO_OWNER,
                        repo: REPO_NAME,
                        accessToken: GITHUB_TOKEN,
                        title: "Update Jenkins Helm chart to version ${latestVersion}",
                        headBranch: BRANCH_NAME,
                        baseBranch: 'main',
                        body: "This PR updates the Jenkins Helm chart dependency to version ${latestVersion} and increments the chart version to ${newChartVersion}."
                    ])
                }
            }
        }
    }
}
