pipeline {
    agent {
        docker {
            image 'moralerr/examples:jenkins-admin-agent-latest'
            label 'standalone'
        }
    }
    environment {
        // Make sure this credential ID has push access to the moralerr/k3s-ansible repo
        GIT_CREDENTIALS = 'GITHUB_ADMIN_TOKEN_AS_PASS'
        GIT_BRANCH = 'master'
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
                        branch: GIT_BRANCH,
                        credentialsId: GIT_CREDENTIALS
                    )

                    // 1) Read the entire file as raw text (preserves all lines)
                    def allFilePath = 'inventory/my-cluster/group_vars/all.yml'
                    def allFileContent = readFile file: allFilePath

                    // 2) Extract the current version from the relevant line
                    def currentVersion = ''
                    allFileContent.eachLine { line ->
                        if (line.trim().startsWith('k3s_version:')) {
                            currentVersion = line.split(':')[1].trim()
                        }
                    }

                    echo "Current k3s_version in all.yml: ${currentVersion ?: '(none found)'}"

                    // 3) Compare versions using our helper
                    def compareResult = compareK3sVersions(latestRelease, currentVersion)
                    if (compareResult > 0) {
                        echo 'Newer version found. Updating all.yml...'

                        // 4) Replace only the line that starts with "k3s_version:"
                        //    (?m) multiline mode so '^' matches start of each line
                        String updatedContent = allFileContent.replaceFirst(
                            '(?m)^k3s_version:.*',
                            "k3s_version: ${latestRelease}"
                        )

                        // 5) Write updated content back, preserving all other lines
                        writeFile file: allFilePath, text: updatedContent

                        // 6) Commit and push changes
                        sh 'git config user.name "moralerr"'
                        sh 'git config user.email "moralerrusc@gmail.com"'
                        sh "git add ${allFilePath}"
                        sh "git commit -m 'Update k3s_version to ${latestRelease}'"

                        // If push fails with "could not read Username," we embed creds in URL
                        withCredentials([usernamePassword(
                            credentialsId: env.GIT_CREDENTIALS,
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_PASS'
                        )]) {
                            sh """
                                git remote set-url origin https://${GIT_USER}:${GIT_PASS}@github.com/moralerr/k3s-ansible.git
                                git push origin ${GIT_BRANCH}
                            """
                        }

                        echo "Successfully updated k3s_version to ${latestRelease} and pushed changes."
                    } else {
                        echo 'No update needed. The current version is already up-to-date or newer.'
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}

//
// Compare version strings like 'v1.27.1+k3s1' (SemVer + optional '+k3sN')
// Returns:
//   1 if ver1 > ver2
//   0 if ver1 == ver2
//  -1 if ver1 < ver2
//
def compareK3sVersions(String ver1, String ver2) {
    // If the strings are null or empty, treat them as "0.0.0"
    if (!ver1?.trim()) ver1 = '0.0.0'
    if (!ver2?.trim()) ver2 = '0.0.0'

    // Strip any leading 'v'
    ver1 = ver1.replaceFirst(/^v/, '')
    ver2 = ver2.replaceFirst(/^v/, '')

    // Remove '+k3s...' if present
    ver1 = ver1.split('\\+')[0]
    ver2 = ver2.split('\\+')[0]

    // Compare major/minor/patch
    def parts1 = ver1.split('\\.')
    def parts2 = ver2.split('\\.')
    int length = Math.max(parts1.size(), parts2.size())

    for (int i = 0; i < length; i++) {
        // Gracefully handle non-numeric parts
        def v1 = (i < parts1.size() && parts1[i].isInteger()) ? parts1[i].toInteger() : 0
        def v2 = (i < parts2.size() && parts2[i].isInteger()) ? parts2[i].toInteger() : 0

        if (v1 > v2) return 1
        if (v1 < v2) return -1
    }
    return 0
}
