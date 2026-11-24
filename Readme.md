# Automated Code Testing and Deployment Pipeline (Jenkins CI/CD)

A Jenkins pipeline that automatically builds, tests and deploys a Java application whenever code is pushed to GitHub. Tests run with JUnit, and the build is deployed to an AWS EC2 (Ubuntu) server only if all tests pass.

## Pipeline Flow

```
git push -> GitHub -> Jenkins detects change -> Checkout -> Build -> JUnit Tests -> Package -> Deploy to AWS EC2
                                                                        |
                                                              (tests fail = pipeline stops, no deployment)
```

## Tools Used

| Tool | Purpose |
|---|---|
| Jenkins (Docker container) | CI/CD server |
| Git and GitHub | Version control and trigger source |
| Maven | Build and dependency management |
| JUnit 5 | Automated test cases |
| AWS EC2 (Ubuntu 22.04) | Deployment (staging) server |
| SSH Agent plugin | Secure deployment from Jenkins to EC2 |

## Project Structure

```
jenkins-demo/
├── Jenkinsfile                      # Pipeline as code
├── pom.xml                          # Maven build configuration
├── README.md
└── src/
    ├── main/java/com/demo/App.java      # Application code
    └── test/java/com/demo/AppTest.java  # JUnit test
```

## Assignment Guidelines Covered

| # | Guideline | How it is done |
|---|---|---|
| 1 | Install Jenkins on a local machine or server | Jenkins runs in a Docker container (`jenkins/jenkins:lts-jdk17`) on an Ubuntu VM |
| 2 | Integrate Git with Jenkins | Pipeline job uses "Pipeline script from SCM" with this GitHub repo and a stored credential (`github-creds`) |
| 3 | Script that triggers on code push | `triggers` block in the `Jenkinsfile` (`githubPush()` and `pollSCM`) plus Poll SCM in the job |
| 4 | Integrate JUnit for automated tests | `mvn test` runs `AppTest`; the `junit` step publishes results in Jenkins |
| 5 | Deploy to AWS | Built jar is copied to an Ubuntu EC2 instance over SSH and executed there |

## Pipeline Stages

1. **Checkout**: pulls the latest code from GitHub.
2. **Build**: `mvn clean compile`.
3. **Test**: `mvn test`, then publishes JUnit reports.
4. **Package**: `mvn package -DskipTests` and archives `target/myapp.jar`.
5. **Deploy to Staging**: copies the jar to the EC2 server with `scp` and runs it with `java -jar` via `ssh`.

Stages run in order, so a failing test stops the pipeline before deployment.

## Setup Guide

### 1. Run Jenkins in Docker

```bash
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  --restart unless-stopped \
  jenkins/jenkins:lts-jdk17
```

Get the initial admin password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open `http://localhost:8080`, install the suggested plugins, and create an admin user.

### 2. Install plugins

GitHub Integration, Pipeline, JUnit, SSH Agent (*Manage Jenkins > Plugins*).

### 3. Configure Maven

*Manage Jenkins > Tools > Maven installations*: name `Maven3`, tick **Install automatically**.

### 4. Add credentials (*Manage Jenkins > Credentials*)

| ID | Kind | Used for |
|---|---|---|
| `github-creds` | Username with password (GitHub username + Personal Access Token) | Cloning the repo |
| `ec2-ssh-key` | SSH Username with private key (user `ubuntu`) | Deploying to EC2 |

Secrets are stored only in Jenkins and are never committed to this repository.

### 5. Create the pipeline job

1. *New Item* > name `my-pipeline` > **Pipeline**.
2. **Build Triggers**: tick **Poll SCM** with schedule `H/2 * * * *`.
3. **Pipeline**: Definition **Pipeline script from SCM**, SCM **Git**.
4. Repository URL: `https://github.com/viraj-vg/jenkins-demo.git`, credentials `github-creds`.
5. Branch `*/main`, Script Path `Jenkinsfile`.
6. Save and click **Build Now**.

### 6. Prepare the AWS EC2 server

1. Launch an **Ubuntu Server 22.04** instance (`t2.micro`) with a key pair (`.pem`).
2. Open port **22** in the Security Group.
3. Install Java on the instance:

```bash
sudo apt update
sudo apt install -y openjdk-17-jre-headless
```

4. Put the instance's public IP in the deploy stage of the `Jenkinsfile`.

## How the Automatic Trigger Works

Jenkins checks GitHub about every 2 minutes (`H/2 * * * *`). When it finds a new commit on `main`, a build starts with no manual action. The `Jenkinsfile` also declares `githubPush()`, so a GitHub webhook can trigger builds instantly when Jenkins is publicly reachable. On a private VM, Poll SCM is used instead.

## Verify a Deployment

On the EC2 server:

```bash
ls -l /home/ubuntu/myapp.jar
java -jar /home/ubuntu/myapp.jar
```

Expected output:

```
Hello from Jenkins CI/CD! 2+3=5
```

## Notes

- If the EC2 instance is stopped and started again, its public IP changes. Update the IP in the `Jenkinsfile` deploy stage.
- Stop or terminate the EC2 instance when not in use to avoid AWS charges.
- For Play Store / App Store apps, the same pipeline can deploy using **fastlane**, with signing keys stored as Jenkins credentials.

## Key Learning

Automating testing and deployment gives faster, repeatable releases with minimal manual intervention: every push is built, tested and, if green, deployed.

