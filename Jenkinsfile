@Library('pipeline-commons') _

pipeline {
    agent {
        docker {
            image 'moralerr/examples:jenkins-admin-agent-latest'
            label 'standalone'
        }
    }
    environment {
        // Retrieve secret values from Jenkins credentials (set these up in Jenkins)
        MASTER_IP     = credentials('MASTER_IP')
        NODE_IP1      = credentials('NODE_IP1')
        NODE_IP2      = credentials('NODE_IP2')
        STANDALONE_IP = credentials('STANDALONE_IP')
        K3S_TOKEN     = credentials('K3S_TOKEN')
    }
    stages {
        stage('Inject Inventory and Group Vars') {
            steps {
                sh '''
                    echo "Injecting IP addresses into inventory files..."
                    sed -i "s/{{ MASTER_IP }}/$MASTER_IP/g" inventory/my-cluster/hosts.ini
                    sed -i "s/{{ NODE_IP1 }}/$NODE_IP1/g" inventory/my-cluster/hosts.ini
                    sed -i "s/{{ NODE_IP2 }}/$NODE_IP2/g" inventory/my-cluster/hosts.ini
                    sed -i "s/{{ STANDALONE_IP }}/$STANDALONE_IP/g" inventory/my-cluster/standalone-host.ini

                    echo "Injecting K3S_TOKEN into group_vars/all.yml..."
                    sed -i 's/{{ K3S_TOKEN }}/'"$K3S_TOKEN"'/g' inventory/my-cluster/group_vars/all.yml

                    echo "Updated hosts.ini:"
                    cat inventory/my-cluster/hosts.ini
                    echo "Updated standalone-host.ini:"
                    cat inventory/my-cluster/standalone-host.ini
                    echo "Updated group_vars/all.yml:"
                    cat inventory/my-cluster/group_vars/all.yml
        '''
            }
        }
        stage('Run Ansible Playbook') {
            steps {
                // sh '''
                //     echo "Running Ansible playbook..."
                //     # For example, use the hosts.ini inventory (adjust as necessary)
                //     ansible-playbook -i inventory/my-cluster/hosts.ini playbooks/site.yml
                // '''
                echo 'Running Ansible playbook...'
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
