pipeline {
    agent any
    
    stages {
        
        stage('GetCode') {
            steps {
                // Limpiamos previamente el directorio de trabajo
                deleteDir()
                // Obtener código del repositorio
                // sh '''
                //     echo "GetCodeMethod: SSH"
                // ''' 
                // git branch: 'develop', credentialsId: '0003dcdb-2921-4880-8ea9-2b344f4778b8', url: 'https://github.com/aga-unir/todo-list-aws.git'
                // INFO: Configuración deprecada, problemas con stage Promote y el uso de la clave privada SSH
                    // SOLUCION: configurar ssh-agent
                    // ALTERNATIVA: uso de GitHub Token (se almacena como parte de la url del repositorio, git config --list)
                withCredentials([string(credentialsId: 'TodoListAws-GitHubURL', variable: 'GitHubURL')]) {
                    sh '''
                        echo "GetCodeMethod: TOKEN"
                        git clone ${GitHubURL} .
                        git checkout develop
                    ''' 
                    sh '''
                        echo "GetRepoConfig: STAGE"
                        wget https://raw.githubusercontent.com/aga-unir/todo-list-aws-config/staging/samconfig.toml -O samconfig.toml-template
                        cat samconfig.toml-template
                    ''' 
                }

            }
        }
        
        stage('Static'){
            steps{
              sh '''
                flake8 --format=pylint --exit-zero src  >flake8.out
                bandit --exit-zero -r src -f custom -o bandit.out --severity-level medium --msg-template "{abspath}:{line}: {severity}: {test_id}: {msg}"
              '''
              recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]
              recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
            }
        }
                   
        
        stage ( 'Deploy'){
          steps{
            sh '''
              echo "INFO: RunSam"
              
              sam build
              sam deploy --resolve-s3 --stack-name "todo-list-aws-staging" --region "us-east-1" --no-fail-on-empty-changeset --parameter-overrides Stage=staging
            '''
            sh'''
              echo "INFO: GetBaseUrlApi and CreateSetEnvBaseUrlApi"
              
              BASE_URL=$(aws cloudformation describe-stacks --stack-name todo-list-aws-staging --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" --output text )
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
              pytest --junitxml=result-integration.xml test/integration/todoApiTest.py || true
            '''            
            
            junit allowEmptyResults: true, testResults:'result-integration.xml'
            script{
              if (currentBuild.result == 'UNSTABLE')
                error("")
            }
            
          }
          
        }
        
        stage ('Promote'){
          steps {
            sh '''
              git checkout master
              git config merge.ours.driver true
              git merge develop --no-edit
              git push
            '''
          }          
        }
        
    }
}
