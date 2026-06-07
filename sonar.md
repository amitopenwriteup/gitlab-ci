# Lab Guide: SonarQube Cloud Setup with GitHub & Jenkins

## Objective

Sign up for a SonarQube Cloud account using GitHub and integrate it with a Jenkins pipeline to analyze Java code quality.

---

## Part 1: Creating a SonarQube Cloud Account via GitHub

### Step 1: Sign Up

1. Go to [https://www.sonarsource.com/products/sonarcloud/](https://www.sonarsource.com/products/sonarcloud/)
2. Click **Sign up** and choose **Sign up with GitHub**
3. Authorize SonarQube Cloud to access your GitHub account

### Step 2: Install SonarQube Cloud GitHub App

4. When prompted, install the SonarQube Cloud GitHub App on your GitHub account
5. Select **Only selected repositories**
6. Choose your target repository by name
7. Click **Save**

### Step 3: Create an Organization

8. After authorization, choose **Create an organization manually**
9. Provide:
   - A **unique organization name**
   - A **unique key**
   - Select the **Free plan**
10. Click **Create Organization**

### Step 4: Set Up Your Project

11. Select your repository from the list
12. Click the **Set Up** button
13. Update the **project key** if needed (make note of this — you'll need it in your pipeline)

---

## Part 2: Configure SonarQube Cloud for Jenkins Integration

### Step 5: Disable Automatic Analysis

Before running a Sonar scan through Jenkins, automatic analysis must be disabled.

1. Go to your SonarQube Cloud project
2. Click **Administration** → **Analysis Method**
3. Toggle off **Automatic Analysis**

### Step 6: Create a Quality Gate

1. Go to your **Organization Settings**
2. Click **Quality Gates** → **Create**
3. Give your Quality Gate a meaningful name (e.g., `Custom Quality Gate`)
4. Click **Add Condition** and configure thresholds as needed (e.g., coverage, duplications, bugs)
5. Navigate to your project → **Administration** → **Quality Gate**
6. Select and **map your project** to the new Quality Gate
7. Click **Set as Default** if you want it applied organization-wide

---

## Part 3: Jenkins Pipeline Configuration

### Step 7: Gather Required Credentials

Before creating the pipeline, collect the following from SonarQube Cloud:

| Parameter | Where to Find |
|-----------|--------------|
| `sonar.login` | **My Account** → **Security** → Generate Token |
| `sonar.projectKey` | Project → **Information** |
| `sonar.organization` | Organization → **Settings** |

> **Security Note:** Never hard-code your token in the pipeline script. Use Jenkins Credentials (Secret Text) and reference it via `withCredentials` or environment variables in production.

### Step 8: Create a Jenkins Pipeline Project

Create a new **Pipeline** job in Jenkins and use the following script. **Replace the placeholders** with your actual values:

```groovy
pipeline {
  agent any

  stages {

    stage('Checkout SCM') {
      steps {
        checkout scmGit(
          branches: [[name: '*/master']],
          extensions: [],
          userRemoteConfigs: [[url: 'https://github.com/amitopenwriteup/sample-java-sonar.git']]
        )
      }
    }

    stage('Compile and Run Tests') {
      steps {
        sh 'mvn clean'
        sh 'mvn compile'
        sh 'mvn test'
      }
    }

    stage('Generate Cucumber Reports') {
      steps {
        script {
          sh 'mvn verify'
        }
      }
    }

    stage('Create Package') {
      steps {
        sh 'mvn package'
      }
    }

    stage('Generate Report') {
      steps {
        sh 'mvn verify'
      }
    }

    stage('Install SonarQube CLI') {
      steps {
        sh 'wget -O sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.7.0.2747-linux.zip'
        sh 'unzip -o -q sonar-scanner.zip'
        sh 'sudo rm -rf /opt/sonar-scanner'
        sh 'sudo mv --force sonar-scanner-4.7.0.2747-linux /opt/sonar-scanner'
        sh 'sudo sh -c \'echo "#/bin/bash \nexport PATH=\\\"$PATH:/opt/sonar-scanner/bin\\\"" > /etc/profile.d/sonar-scanner.sh\''
        sh 'chmod +x /opt/sonar-scanner/bin/sonar-scanner'
        sh '. /etc/profile.d/sonar-scanner.sh'
      }
    }

    stage('Analyze Code Quality') {
      steps {
        sh '''/opt/sonar-scanner/bin/sonar-scanner \
          -Dsonar.projectKey=YOUR_PROJECT_KEY \
          -Dsonar.organization=YOUR_ORGANIZATION_KEY \
          -Dsonar.qualitygate.wait=true \
          -Dsonar.qualitygate.timeout=300 \
          -Dsonar.sources=src/main/java/ \
          -Dsonar.java.binaries=target/classes \
          -Dsonar.host.url=https://sonarcloud.io \
          -Dsonar.login=YOUR_SONAR_TOKEN'''
      }
    }

  }
}
```

> **Replace the following placeholders before running:**
> - `YOUR_PROJECT_KEY` — e.g., `amitopenwriteup_sample-java-sonar`
> - `YOUR_ORGANIZATION_KEY` — e.g., `amitopenwriteup`
> - `YOUR_SONAR_TOKEN` — your SonarQube Cloud token from **My Account → Security**

---

## Reference: SonarQube Scanner Download

The scanner binary used in this lab:

```
https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.7.0.2747-linux.zip
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Quality Gate times out | Increase `-Dsonar.qualitygate.timeout` value |
| `sonar-scanner: command not found` | Verify the install step ran; source the profile script manually |
| Authentication error | Regenerate the token in SonarQube Cloud → **My Account → Security** |
| Automatic analysis conflicts | Ensure automatic analysis is **disabled** in Administration → Analysis Method |

---

## Summary

| Step | Action |
|------|--------|
| 1 | Sign up at SonarQube Cloud using GitHub |
| 2 | Install the GitHub App on your selected repository |
| 3 | Create an organization (free plan) |
| 4 | Disable automatic analysis |
| 5 | Create and configure a Quality Gate |
| 6 | Add the Jenkins pipeline with your token, project key, and org |
| 7 | Run the pipeline and view results in SonarQube Cloud |
