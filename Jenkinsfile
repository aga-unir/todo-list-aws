pipeline {
    agent any
    
    stages {
        
        stage('GetCode') {
            steps {
                // Limpiamos previamente el directorio de trabajo
                deleteDir()
                // Obtener código del repositorio
                sh '''
                    echo "GetCodeMethod: SSH"
                ''' 
                git branch: 'master', credentialsId: '0003dcdb-2921-4880-8ea9-2b344f4778b8', url: 'https://github.com/aga-unir/todo-list-aws.git'
                sh '''
                    echo "GetRepoConfig: PRO"
                    wget https://raw.githubusercontent.com/aga-unir/todo-list-aws-config/production/samconfig.toml
                ''' 
                
            }
        }
        
        stage ( 'Deploy'){
          steps{
            sh '''
              echo "INFO: RunSam"
              
              sam build
              sam deploy --resolve-s3 --stack-name "todo-list-aws-production" --region "us-east-1" --no-fail-on-empty-changeset --parameter-overrides Stage=production
            '''
            sh'''
              echo "INFO: GetBaseUrlApi and CreateSetEnvBaseUrlApi"
              
              BASE_URL=$(aws cloudformation describe-stacks --stack-name todo-list-aws-production --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" --output text )
              echo "INFO: BaseApiUrl=${BASE_URL}"           
              echo "export BASE_URL=${BASE_URL}" > setenv-BaseUrlApi.sh
              chmod a+x setenv-BaseUrlApi.sh
              echo "INFO: Created setenv-BaseUrlApi.sh (útil en un pipeline multi-agente)"
            '''
          }
        }
        
        stage ('Rest Test'){
          steps{
            sh '''
              . ./setenv-BaseUrlApi.sh
              # Aseguramos un "exit cero", con un "o true"
              pytest -m smoke --junitxml=result-integration.xml test/integration/todoApiTest.py || true
            '''            
            
            junit allowEmptyResults: true, testResults:'result-integration.xml'
            script{
              if (currentBuild.result == 'UNSTABLE')
                error("")
            }
            
          }
          
        }
        
        
    }
}
