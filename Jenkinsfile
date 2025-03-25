
pipeline {
    agent any
    
    environment {
        SERVER_IP = '20.198.19.87'
        SSH_USERNAME = 'mithilastackserver'
        DOCKER_TAG = "${BUILD_NUMBER}"
        APP_DIR = "mailcow" // Directory where the application is stored on the server
        APP_NAME = "mailcow" // Name of the applicaton same as in the .env file and in env make this as same github repo name
        PATH = "/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:$PATH"
        REMOTE_DIR = "/home/${SSH_USERNAME}/${APP_DIR}/${APP_NAME}/"
        DEPLOY_DIR = "/home/${SSH_USERNAME}/${APP_DIR}/${APP_NAME}/"
        DEPLOY_TIMEOUT = '3000' // 5 minutes timeout for deployment
        HEALTH_CHECK_RETRIES = '5'
        HEALTH_CHECK_INTERVAL = '10'
    }
    
    
    stages {

        stage('Deploy to Production') {
            steps {
                timeout(time: env.DEPLOY_TIMEOUT.toInteger(), unit: 'SECONDS') {
                    script {
                        try {
                            echo "Cleaning up previous deployment artifacts..."
                            sh "rm -f deploy.tar.gz || true"
                            
                            echo "Copying deployment files to server..."
                            sh '''
                                # Clean any existing archives
                                rm -f deploy.tar.gz || true
                                
                                # Create a list of files to include
                                echo "Creating file list..."
                                find . -type f \
                                    ! -path "./node_modules/*" \
                                    ! -path "./.git/*" \
                                    ! -path "./deploy.tar.gz" \
                                    ! -name "*.log" \
                                    ! -name "*.tmp" \
                                    ! -path "./.next/cache/*" \
                                    > files_to_include.txt

                                # Show what we're going to package
                                echo "Files to be included:"

                                echo "Creating deployment package from workspace..."
                                tar czf deploy.tar.gz --no-recursion --files-from files_to_include.txt || {
                                    echo "Failed to create tar archive"
                                    rm -f files_to_include.txt
                                    exit 1
                                }

                                # Cleanup file list
                                rm -f files_to_include.txt

                                # Verify tar was created successfully and check size
                                if [ ! -f deploy.tar.gz ]; then
                                    echo "Failed to create deployment package!"
                                    exit 1
                                fi
                                echo "Deployment package created successfully"

                                # Check tar file size
                                echo "Checking archive size..."

                                # Test the archive
                                echo "Testing archive integrity..."
                                tar tzf deploy.tar.gz > /dev/null 2>&1 || {
                                    echo "Archive integrity test failed!"
                                    exit 1
                                }
                                 

                                echo "Archive created and verified successfully"

                                # Generate checksum for the deployment package
                                echo "Generating checksum for deployment package..."
                                CHECKSUM=$(sha256sum deploy.tar.gz | awk '{print $1}')
                                echo "$CHECKSUM" | tee deploy.tar.gz.sha256
                                echo "Generated checksum: $CHECKSUM"
                            '''

                            sh """
                                echo "Starting deployment to remote server..."
                                
                                # Prepare remote directory quietly
                                ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ${SSH_USERNAME}@${SERVER_IP} "sudo rm -rf ${REMOTE_DIR}* > /dev/null 2>&1"
                                ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ${SSH_USERNAME}@${SERVER_IP} "sudo mkdir -p ${REMOTE_DIR} > /dev/null 2>&1 && sudo chown ${SSH_USERNAME}:${SSH_USERNAME} ${REMOTE_DIR}"
                                
                                # Copy files quietly
                                echo "Copying deployment files..."
                                scp -q -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa deploy.tar.gz deploy.tar.gz.sha256 ${SSH_USERNAME}@${SERVER_IP}:${REMOTE_DIR}
                                
                                # Execute deployment
                                ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ${SSH_USERNAME}@${SERVER_IP} 'bash -s' << 'ENDSSH'
                                    set -e
                                    cd ${REMOTE_DIR}
                                    
                                    # Extract quietly
                                    tar xzf deploy.tar.gz > /dev/null 2>&1
                                    rm -f deploy.tar.gz deploy.tar.gz.sha256 > /dev/null 2>&1
                                    echo "Here is the present working directory and the files in it"
                                    pwd && ls -la


                                    
                                    cd ${DEPLOY_DIR}
                                    echo " the present working directory and the files in it"
                                    pwd && ls -la
                                    
                                    echo "Checking if docker-compose.yml exists"
                                    if [ ! -f "docker-compose.yml" ]; then
                                        echo "Error: docker-compose.yml not found!"
                                        exit 1
                                    fi
                                    
                                    # Get deployment color
                                    CURRENT_COLOR=\$(sudo docker ps --filter name=${APP_NAME}- --format '{{.Names}}' | grep -oE 'blue|green' | head -n1)
                                    CURRENT_COLOR=\${CURRENT_COLOR:-blue}
                                    NEW_COLOR=\$([ "\$CURRENT_COLOR" = "blue" ] && echo "green" || echo "blue")
                                    
                                    echo "Deploying version ${BUILD_NUMBER} (\$NEW_COLOR) of ${APP_NAME}"
                                    
                                    # Export variables
                                    export COLOR="\$NEW_COLOR"
                                    export TAG="${BUILD_NUMBER}"

                                    # Build and deploy with optimizations
                                    echo "Building new images for ${APP_NAME} ${BUILD_NUMBER} (\$NEW_COLOR)" 
                                    sudo -E docker-compose build --parallel --force-rm --no-cache
                                    echo "Starting new containers..for ${APP_NAME} ${BUILD_NUMBER} (\$NEW_COLOR)"
                                    sudo -E docker-compose up -d --force-recreate --remove-orphans --renew-anon-volumes > /dev/null 2>&1
                                    
                                    echo "✓ Deployment successful - of ${APP_NAME} Version ${BUILD_NUMBER} is running"
ENDSSH
                            """
                        } catch (Exception e) {
                            echo "Deployment failed: ${e.getMessage()}"
                            currentBuild.result = 'FAILURE'
                            throw e
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                def status = currentBuild.result ?: 'SUCCESS'
                def color = status == 'SUCCESS' ? 'green' : 'red'
                
                echo "Pipeline finished with status: ${status}"
                FROM_EMAIL = 'mithilastackclient@gmail.com'
                
                emailext(
                    subject: "[$status] Pipeline: ${currentBuild.fullDisplayName}",
                    body: """
                        <html>
                        <body style="font-family: Arial, sans-serif;">
                            <div style="background-color: ${color}; color: white; padding: 10px; text-align: center;">
                                <h2>Build ${BUILD_NUMBER} - ${status}</h2>
                            </div>
                            <div style="padding: 20px;">
                                <p><strong>Build URL:</strong> <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                                <p><strong> Use the given credentials to access jenkins server: User id: team_mithilastack and Password: team_Mithilastack@12112024</strong></p>
                                <p><strong>Changes:</strong></p>
                                <pre>${currentBuild.changeSets}</pre>
                                <p><strong>Console Output:</strong> Check build page for details</p>
                            </div>
                            <div style="background-color: #f0f0f0; padding: 10px; text-align: center;">
                                <p>Mithilastack DevOps Team</p>
                            </div>
                        </body>
                        </html>
                    """,
                    to: 'itsonlykartik@gmail.com, games.princeraj@gmail.com, meetwithshubhamsharma@gmail.com',
                    from: 'mithilastackclient@gmail.com',
                    replyTo: 'mithilastackclient@gmail.com',
                    mimeType: 'text/html'
                )
                
                echo "Notification email sent"
            }
        }
        
        success {
            echo "Pipeline completed successfully! All stages passed."
        }

        failure {
            echo "Pipeline failed! Please check the logs for details."
        }
    }
}
