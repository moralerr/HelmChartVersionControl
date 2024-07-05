# HelmChartVersionControl

HelmChartVersionControl is a Jenkins pipeline project designed to automate the process of updating the Jenkins Helm chart version in your GitHub repository. This project checks for the latest version of the Jenkins Helm chart, compares it with the Jenkins dependency version in your `Chart.yaml`, updates the file if a newer version is available, increments the chart version, and creates a pull request with the changes.

## Prerequisites

- Jenkins
- GitHub repository access token
- `pipeline-commons` library

## Setup

1. **Clone the Repository**

   ```bash
   git clone https://github.com/moralerr/HelmChartVersionControl.git
   cd HelmChartVersionControl
   ```

2. **Configure Jenkins**

   Ensure that Jenkins has the necessary credentials and access tokens set up. The GitHub access token should be stored in Jenkins credentials with the ID `GITHUB_TOKEN`.

3. **Library Configuration**

   Make sure the `pipeline-commons` library is available to your Jenkins instance. The `utils.groovy` file contains various utility functions used in the Jenkins pipeline.

## Jenkinsfile

The Jenkins pipeline is defined in the `Jenkinsfile`. It uses the `pipeline-commons` library for various utility functions.

### Pipeline Stages

1. **Checkout**

   Clones the GitHub repository and checks out the main branch.

2. **Get Latest Jenkins Helm Chart Version**

   Fetches the latest Jenkins Helm chart version from the GitHub releases.

3. **Get Current Helm Chart Info**

   Retrieves the current Helm chart version and the Jenkins dependency version from `Chart.yaml`.

4. **Update Chart Info**

   If the latest version is greater than the current Jenkins dependency version, it increments the minor version of the chart, resets the patch version, and updates the Jenkins dependency version.

5. **Create Pull Request**

   Creates a pull request with the updated `Chart.yaml` if a newer version is available.

### Schedule

The pipeline is scheduled to run daily at 1 am using a cron trigger.

### Jenkinsfile

```groovy
@Library('pipeline-commons@add-more-common-functions') _
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
```

## Contributing

Contributions are welcome! Please open an issue or submit a pull request with your changes.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.