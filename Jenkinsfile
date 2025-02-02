pipeline {
    agent {
        docker {
            image 'moralerr/examples:jenkins-admin-agent-latest'
            label 'standalone'
        }
    }
    environment {
        // Replace with your actual Jenkins credential ID that has push rights
        GIT_CREDENTIALS = 'GITHUB_ADMIN_TOKEN_AS_PASS'
    }

    stages {
        stage('Check and Update K3s Version') {
            steps {
                script {
                    // STEP 1: Fetch latest release from GitHub API
                    echo 'Fetching latest K3s release...'
                    def response = sh(
                        script: 'curl -s https://api.github.com/repos/k3s-io/k3s/releases/latest',
                        returnStdout: true
                    ).trim()

                    // Parse JSON to get tag_name (e.g., v1.25.4+k3s1)
                    def json = new groovy.json.JsonSlurperClassic().parseText(response)
                    def latestRelease = json.tag_name
                    echo "Latest Release Tag: ${latestRelease}"

                    // STEP 2: Clone moralerr/k3s-ansible repo
                    echo 'Cloning moralerr/k3s-ansible...'
                    git(
                        url: 'https://github.com/moralerr/k3s-ansible.git',
                        branch: 'test',
                        credentialsId: env.GIT_CREDENTIALS
                    )

                    // STEP 3: Read current k3s_version from all.yml
                    def allFilePath = 'inventory/my-cluster/group_vars/all.yml'
                    def allFileContent = readFile file: allFilePath
                    def currentVersion = ""

                    // A simple way to capture the line "k3s_version: ..."
                    allFileContent.eachLine { line ->
                        if (line.trim().startsWith("k3s_version:")) {
                            currentVersion = line.split(":")[1].trim()
                        }
                    }
                    echo "Current k3s_version in all.yml: ${currentVersion}"

                    // STEP 4: Compare versions
                    // Implementing a basic semver-like compare that:
                    // - Strips leading `v`
                    // - Ignores any `+k3sN` suffix for numeric comparison
                    def compareResult = compareK3sVersions(latestRelease, currentVersion)
                    if (compareResult > 0) {
                        // STEP 5: Update and push if the new version is greater
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

                        // Push the change back to the repository
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

//
// Helper method to compare "vX.Y.Z+k3sN" style versions.
//
// Returns:
//  1 if left > right
//  0 if left == right
// -1 if left < right
//
def compareK3sVersions(String ver1, String ver2) {
    // Strip leading 'v'
    ver1 = ver1.replaceFirst(/^v/, "")
    ver2 = ver2.replaceFirst(/^v/, "")

    // Remove any trailing '+...' so we just compare the numeric parts
    ver1 = ver1.split('\\+')[0]
    ver2 = ver2.split('\\+')[0]

    // Split into major/minor/patch
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
