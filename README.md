# Jenkins Pipeline for Apache HTTPD Deployment on EC2

## Prerequisites

1. Two EC2 instances:
   - Jenkins Server: t2.medium (Ubuntu)
   - Target Server: t2.micro (Ubuntu)
2. Security Groups configured:
   - Jenkins: Allow SSH (22) and HTTP (8080)
   - Target: Allow SSH (22) and HTTP (80) from Jenkins IP

## Step 1: Set Up Jenkins Server

### Install Jenkins on t2.medium

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Java 17
sudo apt install openjdk-17-jdk -y

# Add Jenkins repository
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins



# Start Jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins

```

### Access Jenkins
1. Open browser to `http://<jenkins-ip>:8080`
2. Unlock with initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Install suggested plugins
4. Create admin user

## Step 2: Configure Target EC2 (t2.micro)

### Basic Setup
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 -y  # We'll remove this later for Jenkins deployment
```

### Security Group Configuration
1. Go to AWS EC2 Console
2. Select your t2.micro instance
3. Open Security Groups
4. Add inbound rule: HTTP (port 80) from 0.0.0.0/0 (or restrict to Jenkins IP)

## Step 3: Create SSH Credentials in Jenkins

### Diagram of Credential Creation Flow:
```
Jenkins Dashboard → Manage Jenkins → Manage Credentials → System → Global Credentials → Add Credentials
```

### Detailed Steps:
1. Go to Jenkins Dashboard
2. Click "Manage Jenkins" in left sidebar
3. Select "Credentials" under Security
4. Click "System" under Stores scoped to Jenkins
5. Click "Global credentials (unrestricted)"
6. Click "Add Credentials" on left

### Credential Configuration:
- **Kind**: SSH Username with private key
- **Scope**: Global
- **ID**: `httpd-server` (must match pipeline)
- **Description**: "Credentials for Apache target server"
- **Username**: `ubuntu` (or your EC2 username)
- **Private Key**: 
  - Select "Enter directly"
  - Paste your EC2 instance private key (the .pem file contents)
  - OR upload from file

![Credentials Screenshot Description]
(Imagine a screenshot showing the credential configuration form filled as above)

## Step 4: Create Jenkins Pipeline

### Pipeline Configuration:
1. Click "New Item" in Jenkins dashboard
2. Enter name (e.g., "Apache-Deployment")
3. Select "Pipeline" and click OK

### Pipeline Script:
```groovy
pipeline {
    agent any
    
    environment {
        TARGET_SERVER = '18.212.202.80'  // Your target EC2 IP
        SSH_USER = 'ubuntu'               // EC2 username
    }
    
    stages {
        stage('Verify Target Server') {
            steps {
                script {
                    // Verify we can connect before proceeding
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'httpd-server',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            chmod 600 ${SSH_KEY}
                            ssh -i ${SSH_KEY} ${SSH_USER}@${TARGET_SERVER} \
                              'echo "Connection successful to ${TARGET_SERVER}"'
                        """
                    }
                }
            }
        }
        
        stage('Clean Existing Apache') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'httpd-server',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            ssh -i ${SSH_KEY} ${SSH_USER}@${TARGET_SERVER} '
                                sudo systemctl stop apache2 || true
                                sudo apt remove --purge apache2 -y
                                sudo rm -rf /var/www/html/*
                            '
                        """
                    }
                }
            }
        }
        
        stage('Install Apache') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'httpd-server',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            ssh -i ${SSH_KEY} ${SSH_USER}@${TARGET_SERVER} '
                                sudo apt update -y
                                sudo apt install apache2 -y
                                echo "<html>
                                      <body>
                                        <h1>Successfully Deployed via Jenkins!</h1>
                                        <p>Server: ${TARGET_SERVER}</p>
                                        <p>Deployed at: \$(date)</p>
                                      </body>
                                    </html>" | sudo tee /var/www/html/index.html
                                sudo systemctl enable --now apache2
                            '
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'httpd-server',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            # Verify service is running on target
                            ssh -i ${SSH_KEY} ${SSH_USER}@${TARGET_SERVER} '
                                sudo systemctl status apache2
                                curl -I http://localhost
                            '
                            
                            # Verify from Jenkins
                            echo "External verification:"
                            curl -I http://${TARGET_SERVER}
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "Apache successfully deployed to ${TARGET_SERVER}"
            // Optional: Add Slack/email notification here
        }
        failure {
            echo "Deployment failed to ${TARGET_SERVER}"
            // Optional: Add failure notification
        }
    }
}
```

## Step 5: Run the Pipeline

1. Save the pipeline configuration
2. Click "Build Now"
3. Monitor console output:
   - Blue Ocean view provides nice visualization
   - Check each stage execution

## Verification

1. Access your Apache server directly:
   - Open browser to `http://<target-ec2-ip>`
   - Should see "Successfully Deployed via Jenkins!" message

2. Check Jenkins console output for:
   - SSH connection success
   - Package installation logs
   - Service status
   - Curl verification

## Troubleshooting

### Common Issues:
1. **SSH Connection Failed**:
   - Verify security groups allow SSH from Jenkins IP
   - Check key permissions (must be 600)
   - Test SSH manually from Jenkins server

2. **Apache Not Accessible**:
   - Check target security group allows HTTP (port 80)
   - Verify Apache is running:
     ```bash
     sudo systemctl status apache2
     ```

3. **Permission Denied Errors**:
   - Ensure Jenkins user has permission to read the SSH key
   - Check target EC2 user has sudo privileges

## Best Practices

1. **Security**:
   - Restrict security groups to only necessary IPs
   - Rotate SSH keys regularly
   - Use IAM roles instead of keys when possible

2. **Pipeline Improvements**:
   - Add parameterized builds for different environments
   - Implement rollback mechanism
   - Add approval steps for production

3. **Monitoring**:
   - Set up alerts for failed deployments
   - Monitor Apache server metrics
