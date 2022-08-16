#!/usr/bin/env groovy
@Library(['com.optum.jenkins.pipeline.library@master','com.optum.jenkins.pipelines.templates.terraform@master']) _
proceedToBuild = true
pipeline {
    agent {
        label 'docker-maven-slave'
    }
    parameters {
        string(name: 'REPO_NAME', defaultValue: 'MDP_Access_Policy', description: 'Terraform Repository Name')
        choice(choices: ['dev','sit','stg','snd','prod'], description: 'Terraform Environment for deployment', name: 'TERRAFORM_ENVIRONMENT')
        gitParameter branchFilter: 'origin/(.*)', defaultValue: 'master', name: 'BRANCH', type: 'PT_BRANCH'
        string(name: 'APP_NAME', defaultValue: '', description: 'Enter the App team name')
        }
    environment{
        app_name = "${APP_NAME}"
    }
    tools {
        terraform 'Terraform-0_14_6'
    }
    stages{

        stage('Git Checkout')
                {
                    steps{
                        git credentialsId: 'ciapsrvc' , url: 'https://github.optum.com/gov-prog-mdp/MDP_Access_Policy.git' ,  branch: "${params.BRANCH}"
                    }
                }
         stage('Approval'){
           
            when {
               environment name: 'TERRAFORM_ENVIRONMENT', value: 'prod'
               beforeInput true
            }
            
                 steps {
                    script{
                        emailext body:"please check ${BUILD_URL}input to approve or reject. <br>",
                        mimeType: 'text/html',
                        subject: "[Jenkins] ${JOB_NAME} PROD Environment Approval Request",
                        to: "srinivas_cheruvu@optum.com,manish.kumaryadav@optum.com,sudhir_gullapalli@uhc.com"
                        

                        timeout(time: 30, unit: 'MINUTES') {                           
                        input id: 'TERRAFORM_ENVIRONMENT', message: 'Do you want to Deploy to PROD?', ok: 'Deploy', submitter: 'sgullap4,mkumary,scheruv1'
                         }                     
                    }
                }           
        }

        stage('Build Binding Extensions') {
            steps {
                script {
                    
                    sh '''
                    # Run mixins (provides the `dotnet` binary)
                    . /etc/profile.d/jenkins.sh
                    '''
                    def file_var = "${app_name}_environmentvariables.json"
                    def envvariables = readJSON file: "${file_var}"
                    def tfEnv = params.TERRAFORM_ENVIRONMENT
                    switch("${tfEnv}") {
                    case "dev":
                        echo "Hello I am in Dev"
                        env.TFVARS_INIT_PATH = envvariables.dev['TFVARS_INIT_PATH']
                        env.TFVARS_PLAN_APPLY_PATH = envvariables.dev['TFVARS_PLAN_APPLY_PATH']
                        TENV = tfEnv
                        break
                    case "sit":
                        env.TFVARS_INIT_PATH = envvariables.sit['TFVARS_INIT_PATH']
                        env.TFVARS_PLAN_APPLY_PATH = envvariables.sit['TFVARS_PLAN_APPLY_PATH']
                        TENV = tfEnv
                        break
                    case "snd":
                        env.TFVARS_INIT_PATH = envvariables.snd['TFVARS_INIT_PATH']
                        env.TFVARS_PLAN_APPLY_PATH = envvariables.snd['TFVARS_PLAN_APPLY_PATH']
                        TENV = tfEnv
                        break
                    case "stg":
                        env.TFVARS_INIT_PATH = envvariables.stg['TFVARS_INIT_PATH']
                        env.TFVARS_PLAN_APPLY_PATH = envvariables.stg['TFVARS_PLAN_APPLY_PATH']
                        TENV = tfEnv
                        break
                    case "prod":
                        TFVARS_INIT_PATH = envvariables.prod['TFVARS_INIT_PATH']
                        TFVARS_PLAN_APPLY_PATH = envvariables.prod['TFVARS_PLAN_APPLY_PATH']
                        TENV = tfEnv
                        break
                    }
                    env.INIT_PATH = "${TFVARS_INIT_PATH}"
                    env.APPLY_PATH = "${TFVARS_PLAN_APPLY_PATH}"
                    env.TF_ENV = "${TENV}"
                }
            }
        }

        
        /* Terraform Init */
        stage('Terraform Init') {
            steps{
                terraformStep('init', params.TERRAFORM_ENVIRONMENT)
            }
        }
        /* Terraform Validate*/
        stage('Terraform Validate') {
            steps{
                //sh "terraform  validate"
                terraformStep('validate', params.TERRAFORM_ENVIRONMENT)
            }
        }
        /* Terraform Plan */
        stage('Terraform Plan') {
            steps{
                terraformStep('plan', params.TERRAFORM_ENVIRONMENT)
                //approvalWorkflow(params.TERRAFORM_ENVIRONMENT)
            }
        }
        
        /* Terraform Import existing resource into its state */
        stage('Terraform Import') {
            steps{
                terraformStep('import', params.TERRAFORM_ENVIRONMENT)
            }
        } 
        
        /* Terraform Apply*/
        stage('Terraform Apply') {
            when { expression { return proceedToBuild }}
            steps{
                terraformStep('apply', params.TERRAFORM_ENVIRONMENT)
            }
        }
        
        /* Terraform Remove*/
         /* stage('Terraform Remove') {
            steps{
                terraformStep('remove', params.TERRAFORM_ENVIRONMENT)
            }
         } */
        
      
    }
}


def approvalWorkflow(region) {
    echo "Approval"
    emailext body: "Approval Needed for Terraform Apply of " + emailContent + " to $region " +
            "environment. Approval link: ${BUILD_URL}input/",
            subject: "Approval Needed for " + emailContent + " Deployment - $JOB_NAME",
            to: regions[region].EmailNotifyList,
            //cc: uatEmailNotifyList,- Need to check for CC
            from: "noreply@optum.com"

    stage('Deployment Approval') {
        try {
            glApproval message: "Approve deployment to $region?",
                    submitter: regions[region].ApproverList,
                    time: regions[region].ApprovalWaitTime,
                    unit: regions[region].ApprovalWaitTimeUnit
        } catch (e) {
            currentBuild.result = 'SUCCESS'
            proceedToBuild = false
            return
        }
    }
}

def terraformStep(tfStep, tfEnv)
{
    echo "Executing Terraform Step " + tfStep + " on ${tfEnv} :"
    switch (tfEnv) {
        case "dev" :
            credId = "gpd-mdp-sp"
        break
        case "devmdp" :
            credId = "gpd-mdp-sp"
        break
        case "sit" :
            credId = "gpd-mdp-sp"
        break
        case "snd" :
            credId = "gpd-mdp-sp"
        break
        case "stg" :
            credId = "azu-mdp-ciap-cicd-prod"
        break
        case "stgaes" :
            credId = "azu-mdp-ciap-cicd-prod"
        break
        case "prod" :
            credId = "azu-mdp-ciap-cicd-prod"
        break
        default:
            proceedToBuild = false
    }
    echo "Using Credential : " + credId
    stage("Terraform $tfStep"){
        withCredentials([azureServicePrincipal(
                credentialsId: credId,
                subscriptionIdVariable: 'ARM_SUBSCRIPTION_ID',
                clientIdVariable: 'ARM_CLIENT_ID',
                clientSecretVariable: 'ARM_CLIENT_SECRET',
                tenantIdVariable: 'ARM_TENANT_ID'
        )]){
            echo "client ID is $ARM_CLIENT_ID"
            echo "Tenant ID is $ARM_TENANT_ID"
            echo "Client Secret is $ARM_CLIENT_SECRET"
            
            sh "az login --service-principal -u ${ARM_CLIENT_ID} -p ${ARM_CLIENT_SECRET} -t ${ARM_TENANT_ID} --allow-no-subscriptions"
            switch (tfStep) {
                case "init":
                    echo "InitPath is $INIT_PATH" 
                    echo "Executing Terraform init :"
                    sh "ls -la"
                    sh """
                    terraform init -backend-config="${INIT_PATH}"
                    """
                    //sh "terraform  init "
                    break
                case "validate":
                    echo "Executing Terraform Validate :"
                    sh """
                    terraform  validate
                    """
                    break
                case "plan":
                    echo "Executing Terraform plan :"
                    echo "Apply Path is $APPLY_PATH"
                    /*sh '''
                        az login --service-principal -u ${ARM_CLIENT_ID} -p ${ARM_CLIENT_SECRET} -t ${ARM_TENANT_ID} --allow-no-subscriptions
                        az group list --output table > rglist.txt
                        sed -i '1,2d' rglist.txt
                        cat rglist.txt
                        echo "checking for available address space"
                        for rg in $(cat rglist.txt | tr -s ' ' | cut -d ' ' -f1)
                        do
                        az network vnet list --resource-group $rg --subscription ${ARM_SUBSCRIPTION_ID} --output table >> vnetlist.txt
                        done  
                        cat vnetlist.txt 
                        sort vnetlist.txt | uniq >> vnetlist1.txt 
                        awk '{print $5}' vnetlist1.txt >> vnetlist2.txt
                        cat vnetlist2.txt
                        
                        if grep -Fxq "${vnet_uname}" vnetlist2.txt
                        then 
                        echo "Vnet already exists"
                        rm -f temp.txt
                        az network vnet list -g $rsg_name --output table >> temp.txt
                        IP=$(awk 'FNR==3 {print $5}' temp.txt) 
                        echo "IP is $IP"
                        echo "$IP" |awk -F. '{print $1 FS $2}' > substring.txt
                        p1=$(cat substring.txt)
                        echo "p1 is $p1"
                        a1=.0.0/24
                        a2=.1.0/24
                        a3=.2.0/24
                        a4=.2.4
                        a5=.2.5
                        s1=$p1$a1
                        s2=$p1$a2
                        s3=$p1$a3
                        s4=$p1$a4
                        s5=$p1$a5
                        echo "$s1" | tr -d '\n' > ip1.txt
                        echo "$s2" | tr -d '\n' > ip2.txt
                        echo "$s3" | tr -d '\n' > ip3.txt
                        echo "$s4" | tr -d '\n' > ip4.txt
                        echo "$s5" | tr -d '\n' > ip5.txt
                        else 
                            ipstatus=0
                            while [ $ipstatus -le 0 ]
                            do
                            IP=$(printf "10.%d.0.0/16" "$[ $RANDOM % 245 ]")
                            if grep -Fxq "$IP" vnetlist2.txt
                            then
                            echo "Ip found : $IP"
                            ipstatus=1
                            else
                            echo "This Ip not found in assigned IP list"
                            ipstatus=0
                            break
                            fi
                            done  
                            rm -f temp.txt
                            echo "$IP" | tr -d '\n' > temp.txt 
                            echo "$IP" | awk -F. '{print $1 FS $2}' >substring.txt
                            p1=$(cat substring.txt)
                            a1=.0.0/24
                            a2=.1.0/24
                            a3=.2.0/24
                            a4=.2.4
                            a5=.2.5
                            s1=$p1$a1
                            s2=$p1$a2
                            s3=$p1$a3
                            s4=$p1$a4
                            s5=$p1$a5
                            echo "$s1" | tr -d '\n' > ip1.txt
                            echo "$s2" | tr -d '\n' > ip2.txt
                            echo "$s3" | tr -d '\n' > ip3.txt
                            echo "$s4" | tr -d '\n' > ip4.txt
                            echo "$s5" | tr -d '\n' > ip5.txt
                        fi
                    '''
                    IPV = readFile(file: 'temp.txt')
                    IP1 = readFile(file: 'ip1.txt')
                    IP2 = readFile(file: 'ip2.txt')
                    IP3 = readFile(file: 'ip3.txt')
                    IP4 = readFile(file: 'ip4.txt')
                    IP5 = readFile(file: 'ip5.txt')*/

                    /*sh """
                    terraform  plan -out=./output -var-file="${APPLY_PATH}" -var 'client_id="${ARM_CLIENT_ID}"' -var 'client_secret="${ARM_CLIENT_SECRET}"' -var 'tenant_id="${ARM_TENANT_ID}"'\
                        -var 'vnetip=$IPV' \
                        -var 'sprv=$IP1' \
                        -var 'spub=$IP2' \
                        -var 'serv=$IP3' \
                        -var 'dnsr=$IP4' \
                        -var 'dnsr1=$IP5' 
   
                    """*/

                    sh """
                    terraform  plan -out=./output -var-file="${APPLY_PATH}"
                    """
                    break
                case "apply":
                    echo "Executing Terraform apply :"
                    sh """
                     terraform apply -input=false -auto-approve -var-file="${APPLY_PATH}" -var 'client_id="${ARM_CLIENT_ID}"' -var 'client_secret="${ARM_CLIENT_SECRET}"' -var 'tenant_id="${ARM_TENANT_ID}"'
                    """                
                    /*sh """
                    terraform  plan -out=./output -var-file="${APPLY_PATH}" -var 'client_id="${ARM_CLIENT_ID}"' -var 'client_secret="${ARM_CLIENT_SECRET}"' -var 'tenant_id="${ARM_TENANT_ID}"'\
                        -var 'vnetip=$IPV' \
                        -var 'sprv=$IP1' \
                        -var 'spub=$IP2' \
                        -var 'serv=$IP3' \
                        -var 'dnsr=$IP4' \
                        -var 'dnsr1=$IP5' 
   
                    """*/
                    break
                case "import":
                    echo "Executing Terraform import:"
                    sh """
                     
                                        
                    """
                    break 
                case "remove":
                    echo "Executing Terraform remove a resource"
                    sh """
                        
                    """
                    break
                default:
                    proceedToBuild = false
            }

        }
    }
}