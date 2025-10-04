def yc = '/var/lib/jenkins/yc'

pipeline {
    agent any

    environment {
        YC_FOLDER_ID = 'b1gdge57rslfb323otnm'
        YC_CLOUD_ID = 'b1glh5elut8uibsdt45f'
        YC_SERVICE_ACCOUNT_KEY = credentials('iam-key')
    }

    parameters {
        choice(name: 'IMAGE', choices: ['ubuntu-2404-lts', 'debian-12'], description: 'Выберите образ')
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
                        ${yc} config set cloud-id ${YC_CLOUD_ID}
                        ${yc} config set folder-id ${YC_FOLDER_ID}
                        ${yc} config profile activate sa-profile
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
                        ${yc} compute instance create \
                            --name ${vmName} \
                            --zone ru-central1-b \
                            --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
                            --memory ${mem} \
                            --cores ${cpu} \
                            --create-boot-disk image-folder-id=standard-images,image-family=${image},size=${diskSize} \
                            --metadata-from-file user-data=metadata.yaml
                    """

                    timeout(time: 5, unit: 'MINUTES') {
                        waitUntil {
                            def status = sh(
                                script: "${yc} compute instance get --name ${vmName} --format json | jq -r '.status'",
                                returnStdout: true
                            ).trim()
                            echo "VM status: ${status}"
                            return (status == 'RUNNING')
                        }
                    }
                }
            }
        }

        stage('Run Ansible Pipeline') {
            steps {
                script {
                    def vmPubIp = sh(
                        script: "${yc} compute instance get --name ${params.VM_NAME} --format json | jq -r '.network_interfaces[0].primary_v4_address.one_to_one_nat.address'",
                        returnStdout: true
                    ).trim()
                    echo "VM Public IP: ${vmPubIp}"

                    build(
                        job: '345',
                        parameters: [
                            string(name: 'HOST', value: vmPubIp),
                            booleanParam(name: 'CHECK_MODE', value: false)
                        ],
                        wait: true
                    )
                }
            }
        }
    }
}
