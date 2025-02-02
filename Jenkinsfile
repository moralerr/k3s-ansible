pipeline {
    agent {
        docker {
            image 'moralerr/examples:jenkins-admin-agent-latest'
            label 'standalone'
        }
    }
    environment {
        GIT_CREDENTIALS = 'GITHUB_ADMIN_TOKEN_AS_PASS'
    }
    stages {
        stage('Check and Update K3s Version') {
            steps {
                script {
                    echo 'Fetching latest K3s release...'
                    def response = sh(
                        script: 'curl -s https://api.github.com/repos/k3s-io/k3s/releases/latest',
                        returnStdout: true
                    ).trim()
                    def json = new groovy.json.JsonSlurperClassic().parseText(response)
                    def latestRelease = json.tag_name
                    echo "Latest Release Tag: ${latestRelease}"
                    echo 'Cloning moralerr/k3s-ansible...'
                    git(
                        url: 'https://github.com/moralerr/k3s-ansible.git',
                        branch: 'test',
                        credentialsId: env.GIT_CREDENTIALS
                    )
                    def allFilePath = 'inventory/my-cluster/group_vars/all.yml'
                    def allFileContent = readFile file: allFilePath
                    def currentVersion = ''
                    allFileContent.eachLine { line ->
                        if (line.trim().startsWith("k3s_version:")) {
                            currentVersion = line.split(":")[1].trim()
                        }
                    }
                    echo "Current k3s_version in all.yml: ${currentVersion}"
                    def compareResult = compareK3sVersions(latestRelease, currentVersion)
                    if (compareResult > 0) {
                        echo 'Newer version found. Updating all.yml...'
                        def updatedContent = allFileContent.replaceFirst(
                            /(?m)^k3s_version:.*/,
                            "k3s_version: ${latestRelease}"
                        )
                        writeFile file: allFilePath, text: updatedContent
                        sh 'git config user.name "Jenkins"'
                        sh 'git config user.email "jenkins@example.com"'
                        sh "git add ${allFilePath}"
                        sh "git commit -m 'Update k3s_version to ${latestRelease}'"
                        sh 'git push origin test'
                        echo "Successfully updated k3s_version to ${latestRelease} and pushed changes."
                    } else {
                        echo 'No update needed. The current version is already up-to-date or newer.'
                    }
                }
            }
        }
    }
}

def compareK3sVersions(String ver1, String ver2) {
    ver1 = ver1.replaceFirst(/^v/, "")
    ver2 = ver2.replaceFirst(/^v/, "")
    ver1 = ver1.split('\\+')[0]
    ver2 = ver2.split('\\+')[0]
    def parts1 = ver1.split('\\.')
    def parts2 = ver2.split('\\.')
    int length = Math.max(parts1.size(), parts2.size())
    for (int i = 0; i < length; i++) {
        def v1 = (i < parts1.size()) ? parts1[i].toInteger() : 0
        def v2 = (i < parts2.size()) ? parts2[i].toInteger() : 0
        if (v1 > v2) return 1
        if (v1 < v2) return -1
    }
    return 0
}
