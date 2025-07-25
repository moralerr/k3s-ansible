@Library('pipeline-commons') _

pipeline {
    agent {
        kubernetes {
            inheritFrom 'maintenance'
        }
    }
    triggers {
        cron('H H * * *')
    }
    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    parameters {
        choice(
            name: 'PLAYBOOK',
            choices: [
                'playbooks/update_dependencies.yml',
                'site.yml',
                'playbooks/test_ping.yml',
                'playbooks/reboot.yml',
                'playbooks/setup_swarm_agent.yml',
            ],
            description: 'Pick a playbook to run'
        )
        choice(
            name: 'HOSTS_FILE',
            choices: ['inventory/my-cluster/hosts.ini', 'inventory/my-cluster/standalone-host.ini'],
            description: 'Pick a hosts file to use'
        )
        booleanParam(
            name: 'UPGRADE_K3S', 
            defaultValue: false, 
            description: 'Upgrade k3s on the cluster'
        )
    }
    environment {
        // Retrieve secret values from Jenkins credentials (set these up in Jenkins)
        MASTER_IP     = credentials('MASTER_IP')
        NODE_IP1      = credentials('NODE_IP1')
        NODE_IP2      = credentials('NODE_IP2')
        STANDALONE_IP = credentials('STANDALONE_IP')
        K3S_TOKEN     = credentials('K3S_TOKEN')
        ANSIBLE_USER     = credentials('ANSIBLE_USER')
        JENKINS_API_USER = credentials('JENKINS_API_USER')
        JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
    }
    stages {
        stage('Inject Inventory and Group Vars') {
            steps {
                sh 'ansible-galaxy collection install -r ./collections/requirements.yml'
                sh '''
                    echo "Injecting IP addresses into inventory files..."
                    sed -i "s/{{ MASTER_IP }}/$MASTER_IP/g" inventory/my-cluster/hosts.ini
                    sed -i "s/{{ NODE_IP1 }}/$NODE_IP1/g" inventory/my-cluster/hosts.ini
                    sed -i "s/{{ NODE_IP2 }}/$NODE_IP2/g" inventory/my-cluster/hosts.ini
                    sed -i "s/{{ STANDALONE_IP }}/$STANDALONE_IP/g" inventory/my-cluster/standalone-host.ini

                    echo "Injecting K3S_TOKEN into group_vars/all.yml..."
                    sed -i 's/{{ K3S_TOKEN }}/'"$K3S_TOKEN"'/g' inventory/my-cluster/group_vars/all.yml

                    echo "Injecting ANSIBLE_USER into group_vars/all.yml..."
                    sed -i "s/{{ ANSIBLE_USER }}/$ANSIBLE_USER/g" inventory/my-cluster/group_vars/all.yml

                    echo "Intecting Jenkins API secrets into setup_swarm_agent.yml..."
                    sed -i "s/{{ JENKINS_API_USER }}/$JENKINS_API_USER/g" playbooks/setup_swarm_agent.yml
                    sed -i 's/{{ JENKINS_API_TOKEN }}/'"$JENKINS_API_TOKEN"'/g' playbooks/setup_swarm_agent.yml

                    echo "Updated hosts.ini:"
                    cat inventory/my-cluster/hosts.ini
                    echo "Updated standalone-host.ini:"
                    cat inventory/my-cluster/standalone-host.ini
                    echo "Updated group_vars/all.yml:"
                    cat inventory/my-cluster/group_vars/all.yml
                    echo "Updated setup_swarm_agent.yml:"
                    cat playbooks/setup_swarm_agent.yml
                '''
                stash includes: 'inventory/my-cluster/**', name: 'inventory-files'
            }
        }
        stage('Run Ansible Playbook (target -> k3s nodes)') {
            agent {
                docker {
                    image 'moralerr/examples:jenkins-admin-agent-latest'
                    label 'standalone'
                }
            }
            when {
                beforeAgent true
                expression {
                    params.HOSTS_FILE == 'inventory/my-cluster/hosts.ini'
                }
                not { 
                    expression {
                        params.UPGRADE_K3S == true
                    }
                }
            }
            steps {
                unstash 'inventory-files'
                sshagent(credentials: ['SSH_KEY']) {
                    sh """
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        echo "Running Ansible playbook with SSH key loaded..."
                        ansible-playbook -i ${params.HOSTS_FILE} ${params.PLAYBOOK}
                    """
                }
            }
        }
        stage('Run Ansible Playbook (target -> standalone)') {
            when {
                expression {
                    params.HOSTS_FILE == 'inventory/my-cluster/standalone-host.ini'
                }
                not { 
                    expression {
                        params.UPGRADE_K3S == true
                    }
                }
            }
            steps {
                sshagent(credentials: ['SSH_KEY']) {
                    sh """
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        echo "Running Ansible playbook with SSH key loaded..."
                        ansible-playbook -i ${params.HOSTS_FILE} ${params.PLAYBOOK}
                    """
                }
            }
        }
        stage('Upgrade k3s') {
            agent {
                docker {
                    image 'moralerr/examples:jenkins-admin-agent-latest'
                    label 'standalone'
                }
            }
            when {
                beforeAgent true
                anyOf {
                    expression {
                        params.UPGRADE_K3S == true
                    }
                    changeset pattern: 'inventory/my-cluster/group_vars/all.yml', comparator: 'ANT'
                }
            }
            steps {
                unstash 'inventory-files'
                sshagent(credentials: ['SSH_KEY']) {
                    sh '''
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        echo "Running Ansible playbook with SSH key loaded..."
                        ansible-playbook -i inventory/my-cluster/hosts.ini site.yml
                    '''
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
