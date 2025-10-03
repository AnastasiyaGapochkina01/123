def yc = '/var/lib/jenkins/yc'
pipeline {
    agent any

    environment {
        YC_FOLDER_ID = 'b1gdge57rslfb323otnm' 
        YC_CLOUD_ID = 'b1glh5elut8uibsdt45f'
        YC_SERVICE_ACCOUNT_KEY = credentials('key') 
    }

    parameters {
        choice(name: 'IMAGE', choices: ['ubuntu-24-04', 'debian-12'], description: 'Выберите образ')
        string(name: 'CPU', defaultValue: '2', description: 'Количество CPU')
        string(name: 'MEMORY', defaultValue: '4', description: 'Объем памяти в ГБ')
        string(name: 'BOOT_DISK_SIZE', defaultValue: '20', description: 'Размер загрузочного диска в ГБ')
        string(name: 'VM_NAME', defaultValue: 'my-vm', description: 'Имя виртуальной машины')
    }

    stages {
        stage('Setup yc CLI') {
            steps {
                script {
                    sh """
                        ${yc} config profile create sa-profile || true
                        ${yc} config set service-account-key ${env.YC_SERVICE_ACCOUNT_KEY}
                        ${yc} config set cloud-id $YC_CLOUD_ID
                        ${yc} config set folder-id $YC_FOLDER_ID
                        ${yc} config set active sa-profile
                    """
                }
            }
        }

        stage('Create VM') {
            steps {
                script {
                    def vmName = params.VM_NAME
                    def image = params.IMAGE
                    def cpu = params.CPU
                    def mem = params.MEMORY + 'gb'
                    def diskSize = params.BOOT_DISK_SIZE + 'gb'

                    sh """
                    \$yc compute instance create \
                        --name $vmName \
                        --zone ru-central1-b \
                        --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
                        --memory $mem \
                        --cores $cpu \
                        --create-boot-disk image-family=$image,size=$diskSize \
                        --metadata-from-file user-data=metadata.yaml
                    """

                    // Проверяем статус VM до running
                    timeout(time: 5, unit: 'MINUTES') {
                        waitUntil {
                            def status = sh(script: "\$yc compute instance get --name $vmName --format json | jq -r '.status'", returnStdout: true).trim()
                            echo "VM status: ${status}"
                            return (status == 'RUNNING')
                        }
                    }
                }
            }
        }
    }
}
