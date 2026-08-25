# Jenkins Learning Notes

> Simple notes built up as we go, question by question. Written in plain language, no heavy jargon.

## Table of Contents
1. Introduction & Basics
2. Installation & Setup
3. Core Concepts
4. Jobs & Pipelines
5. Plugins & Integrations
6. Advanced Topics
7. Troubleshooting & Tips
8. Reference Files (Full Source)

---

## 1. Introduction & Basics

### 1.1 What is Jenkins?

Jenkins is a tool that **automatically does the boring, repeated steps** of taking your code from "I just wrote this" to "it's tested, packaged, and ready to use" — so you don't have to type all those commands by hand every time.

Imagine every time you finish writing code, you had to manually:
1. Run all the tests, to make sure nothing broke.
2. Package the code properly.
3. Build a Docker image from it.
4. Upload that image somewhere.
5. Put it on a server so people can use it.

Doing this by hand every time is slow and easy to get wrong. Jenkins lets you write these steps down **once**, and then it does them for you every time, the exact same way, without ever forgetting a step.

### 1.2 What is CI/CD?

You'll hear "CI/CD" a lot — it just means:

- **CI (Continuous Integration):** every time someone adds new code, automatically test it right away, instead of finding out something's broken weeks later.
- **CD (Continuous Delivery/Deployment):** once the code passes its tests, automatically get it ready to use (and sometimes put it live), instead of a person doing that by hand.

Jenkins is a tool that helps you do both.

### 1.3 Why use Jenkins?

- It's **free**.
- It has thousands of small add-ons (called **plugins**) that let it connect to almost anything — GitHub, Docker, AWS, email, etc.
- It works with pretty much any programming language.
- It runs on your own server (like your EC2 instance), so you're fully in control of it.

### 1.4 The basic picture

Here's the simplest way to picture the whole system:

1. You write code and push it to **GitHub**.
2. GitHub tells **Jenkins**, "new code just came in" (this is the webhook we set up).
3. Jenkins pulls the new code and runs through a list of steps you wrote ahead of time. This list is called a **Pipeline**, and it's written in a file called a **Jenkinsfile**.
4. Those steps might be: run tests → build the project → build a Docker image → upload it → put it on a server.
5. If any step fails, Jenkins stops and shows you exactly where it failed.

Everything else we learn (jobs, plugins, passwords, servers) exists to support this one loop: **code changes → Jenkins notices → Jenkins runs your steps automatically.**

---

## 2. Installation & Setup

We have two files — a `Dockerfile` and a `docker-compose.yml` — that we upload onto the EC2 server and use to set up and run Jenkins there. This section explains what's inside these files, how to get them onto EC2, how to install Docker, and how to start Jenkins.

### 2.1 Docker CLI vs the Docker engine (an important idea)

Docker is really made of two parts:
- **The Docker engine** — the actual program running in the background that does the real work: builds images, runs containers. This has to be installed and running on some machine.
- **The Docker CLI** — the `docker` command you type. On its own, it doesn't *do* anything — it just sends instructions to the engine and shows you the result.

In our setup, only the **CLI** is installed inside the Jenkins container. There's no full Docker engine running inside that container. Instead, the compose file connects the container to the Docker engine that's already running on the **EC2 machine itself**:
```yaml
- /var/run/docker.sock:/var/run/docker.sock
```
Think of this line like a phone line between the container and the engine sitting outside on the EC2 machine. So when Jenkins runs `docker build`, it's really asking the EC2 machine's own Docker engine to do the work — Jenkins is borrowing it instead of running its own copy.

### 2.2 Setting up Jenkins on EC2 using Docker

We use two files together: a **Dockerfile** (which describes how to build our own custom Jenkins) and a **docker-compose.yml** (which describes how to run it).

#### The Dockerfile, explained

```dockerfile
FROM jenkins/jenkins:lts-jdk21
```
This starts from the official Jenkins image, which already comes with Java built in (Jenkins needs Java to run).

```dockerfile
USER root
```
Switches to the "root" user temporarily, so we're allowed to install extra software.

**Basic tools installed:** things like `git`, `curl`, `wget`, `unzip`, and a few others — small helper programs Jenkins will need while running your pipelines. At the end, the leftover installation files are deleted to keep the image small.

**Docker CLI installed:** adds Docker's official download source and installs just the `docker` command (not the full engine) — as explained above, it'll use the EC2 machine's own Docker engine instead.

**Maven installed:** a tool used to build Java projects.

**kubectl installed:** a tool used to talk to Kubernetes, in case pipelines ever need to deploy there.

```dockerfile
USER jenkins
RUN jenkins-plugin-cli --plugins workflow-aggregator git github credentials-binding ssh-agent docker-workflow junit pipeline-stage-view timestamper
```
Switches back to the regular `jenkins` user (safer than staying as root), and installs a set of add-ons (plugins) right away:

| Plugin | What it does |
|---|---|
| `workflow-aggregator` | The main plugin that lets Jenkins understand Pipelines at all |
| `git` / `github` | Lets Jenkins talk to Git and GitHub |
| `credentials-binding` | Lets Jenkins safely use stored passwords/keys inside a pipeline |
| `ssh-agent` | Lets Jenkins use SSH keys inside a pipeline |
| `docker-workflow` | Lets Jenkins build/run Docker images from a pipeline |
| `junit` | Lets Jenkins show test results nicely |
| `pipeline-stage-view` | Shows each pipeline stage as a box in the Jenkins UI |
| `timestamper` | Adds a timestamp next to every line in the logs |

#### The docker-compose.yml, explained

```yaml
services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
```
Says: build our own Jenkins image using the Dockerfile above, instead of using a plain, ready-made one.

```yaml
    container_name: jenkins
    restart: unless-stopped
```
Names the container `jenkins`. `restart: unless-stopped` means: if it crashes, or if the EC2 machine reboots, automatically start it again — unless a person manually stopped it on purpose.

```yaml
    user: root
```
Runs the container as `root`, mainly so it's allowed to use the shared Docker connection mentioned earlier.

```yaml
    ports:
      - "8080:8080"   # Jenkins website
      - "50000:50000" # used if you add separate helper machines later
```

```yaml
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
```
- `jenkins_home`: a storage folder that keeps all your Jenkins data (jobs, settings, plugins) saved even if the container is deleted and recreated.
- `/var/run/docker.sock`: the "phone line" mentioned earlier, letting the container use the EC2 machine's own Docker engine.

```yaml
    environment:
      JAVA_OPTS: >
        -Djenkins.install.runSetupWizard=true
```
Tells Jenkins to show the first-time setup screen (asking for a password, then letting you install plugins) the first time it starts.

```yaml
volumes:
  jenkins_home:
networks:
  jenkins-network:
    driver: bridge
```
Just declares the storage folder and a small internal network, useful later if you add more containers.

### 2.3 Getting the files onto EC2

A few common ways, easiest first:

1. **Copy them over with `scp`**
   From your own computer (not EC2):
   ```bash
   scp -i /path/to/your-key.pem Dockerfile docker-compose.yml ubuntu@<EC2_PUBLIC_IP>:/home/ubuntu/jenkins-setup/
   ```
   (Make the folder first with `mkdir jenkins-setup` over SSH, or just scp them into `~/`.)

2. **Type/paste them directly on EC2** (no file transfer needed)
   Connect to EC2 with SSH, then:
   ```bash
   mkdir ~/jenkins-setup && cd ~/jenkins-setup
   nano Dockerfile        # paste the content, Ctrl+O to save, Ctrl+X to exit
   nano docker-compose.yml
   ```

3. **Put them in a GitHub repo and download them on EC2** (best if you'll keep changing them over time)
   ```bash
   git clone https://github.com/<you>/<repo>.git
   cd <repo>
   ```

For a one-time setup like this, **option 1 or 2** are the simplest.

### 2.4 Installing Docker on EC2 (Ubuntu)

Connect to your EC2 machine first:
```bash
ssh -i /path/to/your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

**Simple way (recommended):**
```bash
sudo apt update
sudo apt install docker.io docker-compose-v2 -y
```
- `docker.io` → installs the full Docker engine plus the `docker` command, all packaged by Ubuntu itself.
- `docker-compose-v2` → installs the modern `docker compose` tool (matches the `docker compose up` command we use later).

No extra setup needed — this is enough for learning and for most real use.

Start Docker and make sure it turns on automatically every time the machine boots (important so Jenkins comes back after a reboot):
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Let your user run `docker` commands without typing `sudo` every time:
```bash
sudo usermod -aG docker $USER
newgrp docker   # or log out and back in for this to take effect
```

Check it worked:
```bash
docker --version
docker compose version
```

### 2.5 Starting Jenkins and keeping it running

From the folder that has your `Dockerfile` and `docker-compose.yml`:
```bash
docker compose up -d
```
- `up` builds the image and starts it running.
- `-d` means "run in the background" — so it keeps running even after you close your SSH connection.

Check it's running:
```bash
docker compose ps
docker logs -f jenkins   # watch the logs live
```

**Get the first-time password (Jenkins will ask for this when you first open it in your browser — this is the "Unlock Jenkins" screen):**
```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
Copy the exact text this prints out, and paste it into the "Administrator password" box on the website.

**Where does this password come from?** You never set it — Jenkins makes it up itself. The very first time Jenkins starts, it:
1. Creates a random, one-time password.
2. Saves it into a file inside the container.
3. Also prints it into the logs.
4. Locks the website until you type that exact password in — this is just a safety check to prove you actually have access to the server.

This password is only used once. Later in setup, you'll create your real username and password.

**What the command above actually does, piece by piece:**

| Part | Meaning |
|---|---|
| `docker exec` | Run a command **inside a container that's already running** |
| `jenkins` | The name of the container to run it inside — matches `container_name: jenkins` in our compose file |
| `cat /var/jenkins_home/secrets/initialAdminPassword` | The actual command — `cat` just means "print this file's text on screen" |

In plain words: *"go inside the running Jenkins container, and print out the contents of that password file, so I can read it and copy it."*

> **If you see "password entered is incorrect":** run the command again to get the current password — if the container restarted since you first opened the page, a new password may have been made. Also make sure you copied the text exactly, with no extra spaces.

Then open `http://<EC2_PUBLIC_IP>:8080` in your browser (make sure your EC2 **Security Group** allows the outside world to reach port 8080), enter the password, and follow the rest of the setup (choosing plugins, then creating your real admin username and password).

**Will Jenkins come back automatically if the EC2 machine restarts?** Yes — you won't need to run `docker compose up` again. Here's why:
1. `sudo systemctl enable docker` (done earlier) makes Docker itself start automatically whenever EC2 boots.
2. Once Docker starts, it checks each container to see if it should be running.
3. `restart: unless-stopped` in our compose file tells Docker: "always start this one back up" — unless a person had manually stopped it.

So: EC2 restarts → Docker starts itself → Docker sees Jenkins should be running → it starts Jenkins automatically.

**One exception:** if you had manually run `docker compose stop` before the restart, Docker respects that and won't start it again — you'd need to run `docker compose start` yourself.

To stop or restart it later:
```bash
docker compose stop      # stop it, but keep it ready to start again
docker compose start     # start it again
docker compose down      # stop and remove the container (your saved data stays safe)
docker compose up -d --build   # rebuild it if you changed the Dockerfile, then start
```

---

## 3. Core Concepts

Here are the main words you'll keep hearing in Jenkins, explained simply.

### 3.1 Job (also called "Item")

A **Job** is a saved task you create in Jenkins, like "build and test my todo-backend app." You make one job per project. Inside a job, you tell Jenkins where your code is and what steps to run. A **Pipeline** is the most common type of job.

### 3.2 Pipeline

A **Pipeline** is just the ordered list of steps Jenkins should run for your project, like a recipe: "first run the tests, then build the code, then build a Docker image, then upload it." You write this list once, and Jenkins follows it every time.

### 3.3 Jenkinsfile

The **Jenkinsfile** is a plain text file that sits inside your project's own GitHub folder, and it contains your Pipeline written out. Since it lives with your code, any changes to it get tracked in Git, just like any other code change.

### 3.4 Stage

A **Stage** is one labeled section of your Pipeline, like a chapter. For example, "Test," "Build," "Docker Build," and "Docker Push" are separate stages. Jenkins shows each one as its own box, so you can see at a glance which part passed or failed.

### 3.5 Step

A **Step** is one single action inside a stage — the actual command being run. For example, `./mvnw test` is a step. A stage can have one or many steps.

### 3.6 Agent / Node

An **Agent** (also called a **Node**) is the actual machine that runs your pipeline's commands. In our setup we only have one — the "Built-In Node," which is the same EC2 machine Jenkins itself runs on. Bigger setups sometimes add extra separate machines so heavy builds don't slow down Jenkins itself.

### 3.7 Executor

An **Executor** is like an open "slot" on a machine that can run one build at a time. If a machine has 2 executors, it can run 2 builds at once. If all slots are busy, new builds just wait in line.

### 3.8 Plugin

A **Plugin** is a small add-on that gives Jenkins a new ability it doesn't have by default, like talking to GitHub, or running Docker commands. Jenkins by itself is quite basic — nearly everything useful comes from installing the right plugins.

### 3.9 Credentials

**Credentials** are Jenkins' safe storage for sensitive things like passwords and keys, so you never have to type them directly into your Jenkinsfile where anyone could read them. Your Jenkinsfile just refers to a credential by name, and Jenkins fills in the real value while it's running, keeping it hidden from the logs.

### 3.10 Trigger

A **Trigger** is what tells Jenkins "start a new build now." This could be a person clicking a button, a set schedule, or — what we set up — a GitHub notification the moment new code is pushed.

### 3.11 Build

A **Build** is one single run of your pipeline, start to finish. Each one gets a number (`#1`, `#2`, `#3`...) so you can look back at any past run and see exactly what happened.

### 3.12 Workspace

The **Workspace** is the folder where Jenkins actually downloads your code and does its work for a build. Think of it as Jenkins' temporary working folder for your project.

### 3.13 Console Output

The **Console Output** is the full log of everything that happened during a build. It's the first place to check whenever something goes wrong.

---

## 4. Jobs & Pipelines

### 4.1 Example Jenkinsfile, explained line by line

This example is a pipeline for a Java ("todo-backend") app that tests it, builds it, builds a Docker image, and uploads that image.

```groovy
pipeline {
    agent any
```
`agent any` means: run this pipeline on whatever machine is available. Since we only have one machine set up (our EC2 box), this just means "run it here."

```groovy
environment {
    PROJECT_NAME="todo-backend"
    DOCKER_IMAGE="batchlcwd/simple-todo-backend"
    DOCKER_TAG="${BUILD_NUMBER}"
    EC2_HOST="13.204.45.50"
    EC2_USER="ubuntu"
    DOCKER_CONTAINER="todo-backend"
    APP_PORT="8082:8080"
}
```
This sets up some named values that can be reused anywhere in the pipeline. `BUILD_NUMBER` is a special value Jenkins fills in automatically — it goes up by one every time (1, 2, 3...), so each build gets its own unique image label, instead of overwriting the same one every time.

```groovy
stages {
    stage("Checkout") {
        steps {
            checkout scm
            echo "checkout successful"
        }
    }
```
- `stages { }` — the pipeline's steps, one after another, shown as boxes in the Jenkins website.
- `stage("Name")` — one named section.
- `steps { }` — the actual actions inside that section.
- `checkout scm` — downloads your code from GitHub, using whatever repo/branch settings were already set up when the job was created.
- `echo` — just prints a message in the log, useful for keeping track of what's happening.

```groovy
stage("Test") {
    steps {
        sh '''
        chmod +x ./mvnw
        ./mvnw test
        '''
    }
}
```
- `sh` — tells Jenkins to run a command on the machine, exactly like typing it into a terminal.
- `'''...'''` — lets you write several lines of commands inside one `sh` step.
- `chmod +x ./mvnw` — makes a file called `mvnw` runnable (sometimes this permission gets lost when downloading from GitHub).
- `./mvnw test` — runs the project's automated tests.

```groovy
stage("Build") {
    steps {
        sh '''
        ./mvnw clean package -DskipTests
        '''
    }
}
```
- `clean` — deletes any old build files from before.
- `package` — compiles the code and packages it up (into a `.jar` file, for a Java project).
- `-DskipTests` — skip running the tests again here, since they already ran in the step before.

```groovy
stage("Docker Build") {
    steps {
        sh '''
        docker build \
        -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
        '''
    }
}
```
- `docker build` — builds a Docker image using the project's own Dockerfile (a different one from the Jenkins server's own Dockerfile — this one lives with the app's code).
- `-t ${DOCKER_IMAGE}:${DOCKER_TAG}` — gives the image a name and label, using the values we set up earlier.
- `.` — means "use the current folder" as the source for the build.
- The `\` at the end of a line just means "this command continues on the next line."

```groovy
stage("Docker Push") {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'dockerhub-credentials',
                usernameVariable: 'DOCKERHUB_USERNAME',
                passwordVariable: 'DOCKERHUB_PASSWORD'
            )
        ])
        {
            sh '''      
                echo "$DOCKERHUB_PASSWORD" | docker login \
                -u $DOCKERHUB_USERNAME \
                --password-stdin
                docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                docker image prune -af 
                docker images
            '''
        }
    }
}
```
- `withCredentials([...])` — safely fetches a saved password from Jenkins' credential storage, and makes it available (but hidden in the logs) only inside this block.
- `usernamePassword(credentialsId: 'dockerhub-credentials', ...)` — says "get the credential saved under the name `dockerhub-credentials`, and let me use its username/password as `$DOCKERHUB_USERNAME` and `$DOCKERHUB_PASSWORD`."

This step then runs **4 separate commands, one after another**, just like typing them one by one into a terminal:
```bash
echo "$DOCKERHUB_PASSWORD" | docker login -u $DOCKERHUB_USERNAME --password-stdin
docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
docker image prune -af
docker images
```

**Why `echo` is used here:** to upload the image, you first need to log in to Docker Hub. A simple login would look like `docker login -u myuser -p mypassword`, but writing the password directly like that isn't safe — anyone else using that same machine could see it just by looking at the running commands list. So instead, Docker offers a safer way: you can "type" the password into it silently. `echo "$DOCKERHUB_PASSWORD"` just prints the password, and the little `|` symbol (called a "pipe") hands that printed text directly into `docker login`, instead of it ever appearing as a visible part of the command. In short: *print the password, and quietly feed it straight into the login step, instead of showing it anywhere.*

**Why these commands are grouped together:** they need to happen one after another, in order:
1. `docker login ...` — log in to Docker Hub first (this stays "logged in" for the rest of this step).
2. `docker push ...` — upload the image (only works because we just logged in).
3. `docker image prune -af` — delete old, unused images to free up space.
4. `docker images` — just print out what's left, so you can see it in the log.

> Note: `EC2_HOST`, `EC2_USER`, `DOCKER_CONTAINER`, `APP_PORT` are set up but not used yet in this pipeline — they're probably meant for a later step that connects to EC2 and actually runs the uploaded image there.

### 4.2 Creating credentials (saved passwords/keys) in Jenkins

General path: **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

#### Docker Hub login (`dockerhub-credentials`)
- **Kind:** `Username with password`
- **Username:** your Docker Hub username
- **Password:** your Docker Hub password, or better, a **Docker Hub Access Token** (Docker Hub → Account Settings → Security → New Access Token) — this can be turned off later without changing your real password.
- **ID:** `dockerhub-credentials` — this name must match **exactly** what's written in the Jenkinsfile, or Jenkins won't be able to find it.
- **Description:** just for your own notes, e.g. "Docker Hub login."

#### EC2 login (for a future step that deploys to EC2)
EC2 machines usually don't use a username/password — they use a **key file** instead, so this is a different kind of credential:
- **Kind:** `SSH Username with private key`
- **ID:** e.g. `ec2-ssh-credentials`
- **Username:** `ubuntu` (matches `EC2_USER` in the Jenkinsfile — the default username for Ubuntu EC2 machines)
- **Private Key:** choose "Enter directly," then paste the full contents of your `.pem` key file (the same one you use for `ssh -i your-key.pem ubuntu@...`), including the `-----BEGIN...` and `-----END...` lines.
- **Passphrase:** leave empty, unless your key file itself has a passphrase set.

`EC2_HOST="13.204.45.50"` in the Jenkinsfile is just the plain address — no credential needed for that part; it only tells the pipeline *where* to connect, while the key above handles *how* to log in.

### 4.3 Why you need to create a Pipeline job in the Jenkins website

Your `Jenkinsfile` sitting inside a GitHub repo is just a text file — Jenkins doesn't know it exists until you create a **Job** (choosing type "Pipeline") in the Jenkins website that points at it. That job tells Jenkins:
- Which GitHub repo to look at
- Which branch to use
- Where the Jenkinsfile is inside that repo
- What should start a build (a manual click, a schedule, or a GitHub push)

Without this job, even a perfectly written Jenkinsfile will never actually run.

### 4.4 Creating the Pipeline job

1. Jenkins homepage → **New Item**
2. Type a name, choose **Pipeline**, click OK
3. Scroll down to the **Pipeline** section:
   - **Definition:** `Pipeline script from SCM` (this means "go fetch the script from a Git repo," instead of typing it directly here)
   - **SCM:** `Git`
   - **Repository URL:** your GitHub repo's link
   - **Credentials:** only needed if the repo is private — a GitHub access token works well here (added the same way as before)
   - **Branch Specifier:** `*/main` (or whatever your branch is called)
   - **Script Path:** `Jenkinsfile` (leave as-is if it's in the main folder of the repo)
4. Click **Save**

You can click **Build Now** to test it by hand first, before setting up automatic triggering.

### 4.5 What is a webhook, and why use one?

A **webhook** is a way for one system (GitHub) to instantly tell another system (Jenkins) the moment something happens — by sending a small message to a web address you set up.

**The alternative would be "checking in" regularly** — Jenkins could ask GitHub every few minutes, "did anything change?" But that wastes requests (even when nothing changed) and adds delay (you'd have to wait for the next check). A webhook flips this: GitHub tells Jenkins the moment a push happens, so builds start within seconds.

### 4.6 Creating the webhook on GitHub

1. On your repo: **Settings → Webhooks → Add webhook**
2. **Payload URL:** `http://<EC2_PUBLIC_IP>:8080/github-webhook/` — a special web address that Jenkins automatically understands, thanks to the GitHub plugin we already installed. Don't forget the trailing slash.
3. **Content type:** `application/json`
4. **Secret:** optional — adds an extra layer of proof that the message really came from GitHub (fine to skip while learning)
5. **Which events:** choose "Just the push event"
6. Make sure **Active** is checked
7. Click **Add webhook**

GitHub will immediately send a small test message — check "Recent Deliveries" on the same page; a green result means Jenkins received it fine.

### 4.7 Telling the Jenkins job to listen for it

Setting up the webhook isn't enough by itself — you also need to tell the specific job to react to it:
1. Open your Pipeline job → **Configure**
2. Under **Build Triggers**, check ✅ **"GitHub hook trigger for GITScm polling"**
3. Save

### 4.8 The full flow, start to finish

1. You push new code to GitHub.
2. GitHub sends a message to `http://<EC2_PUBLIC_IP>:8080/github-webhook/`.
3. Jenkins receives it and checks which jobs are set up to listen for that repo.
4. It starts a new build of that job.
5. The pipeline runs — `checkout scm` grabs the exact code just pushed — then goes through the rest of the steps automatically.

**One thing to check:** GitHub (out on the internet) needs to be able to reach your EC2 machine on port 8080 for step 2 to work — so your EC2 **Security Group** must allow that port in, same as what lets you open the Jenkins website yourself.

### 4.9 Production Approval stage (a safety checkpoint before deploying)

```groovy
stage("Production Approval "){
    steps{
        input message: 'Deploy to production?'
    }
}
```

This stage simply **pauses the pipeline** and shows a button on the Jenkins website asking "Deploy to production?" Jenkins won't do anything else until a real person clicks "Proceed" (or "Abort" to cancel). It's a safety checkpoint so nothing gets deployed to your live server automatically, without a human saying yes first.

### 4.10 EC2 Deploy stage (actually deploying to the production server)

```groovy
stage("EC2 Deploy- production server")
{
    steps{
        sshagent([
            'ec2-instance-key'
        ]){
            sh '''
            mkdir -p ~/.ssh
            chmod 700 ~/.ssh
            ssh-keyscan -H "$EC2_HOST" >> ~/.ssh/known_hosts
            ssh $EC2_USER@$EC2_HOST "
                docker rm -f $DOCKER_CONTAINER || true
                docker pull $DOCKER_IMAGE:$DOCKER_TAG
                docker run -d \
                    --name $DOCKER_CONTAINER \
                    -p $APP_PORT \
                    --restart unless-stopped \
                    $DOCKER_IMAGE:$DOCKER_TAG
                docker image prune -f
            "
            '''
        }
    }
}
```

- `sshagent(['ec2-instance-key'])` — loads a saved SSH key (from Jenkins credentials, named `ec2-instance-key`) so the commands inside can log into the EC2 machine, without needing a password typed in.

> **Note: in this setup, Jenkins and the app run on the *same* EC2 machine.** `EC2_HOST` in the Jenkinsfile should be set to that same server's address (the `13.204.45.50` shown in the example Jenkinsfile is just a placeholder/typo value — update it to match your real EC2 IP). Even though it's the same physical machine, SSH still treats this as a brand-new connection the first time — it doesn't know or care that it's "the same computer" in a physical sense, so the trust step below is still needed. Writing the deploy step this way (SSH into the target server, rather than running Docker commands directly) also means the exact same Jenkinsfile would keep working unchanged if you ever moved the app to a separate production server later — you'd just update `EC2_HOST`.

- `mkdir -p ~/.ssh` — creates a folder called `.ssh`, where SSH keeps its list of "servers I trust." `mkdir` means "make folder," and `-p` means "don't complain if it already exists." Without this folder, SSH would have nowhere to save that trust-list in the next step.

- `chmod 700 ~/.ssh` — locks down that folder so only the current user can open it. This might seem unusual, but SSH is strict about security: it actually **refuses to work** if it sees that its folder could be opened by other users on the machine — like a bank refusing to use a vault whose door was left open for anyone to see inside. `chmod` means "change who's allowed to access this," and `700` means "only the owner can read, write, or open it — nobody else."

- `ssh-keyscan -H "$EC2_HOST" >> ~/.ssh/known_hosts` — this one needs a small story to make sense. Whenever you connect to **any** server over the internet for the very first time — for any reason, not just deploying — how do you really know you're talking to the actual server you meant to reach, and not some fake pretending to be it? SSH solves this by having every server show a unique "identity card" the first time you connect, and SSH wants confirmation that this really is the server it's supposed to be, before continuing. Normally, a person looks at that identity card and manually types "yes, I trust this" the first time. Since nobody's sitting there to approve it during an automatic pipeline, we pre-approve it ourselves with this line — telling SSH ahead of time, "I already know and trust this specific server's identity, so connect without stopping to ask me."
  - `ssh-keyscan` = a small tool whose only job is "go get a server's identity card"
  - `-H` = hides/scrambles the server's address in the saved file, for extra privacy
  - `"$EC2_HOST"` = the production server's address (e.g. `13.204.45.50`)
  - `>>` = "add this to the end of the file" (instead of replacing everything already in it)
  - `~/.ssh/known_hosts` = the specific file SSH checks to see which servers it already trusts

**In short, these 3 lines exist only to make the real connection (the `ssh $EC2_USER@$EC2_HOST "..."` line right after) run smoothly and automatically, without Jenkins ever getting stuck waiting for a "do you trust this?" prompt that no human is there to answer.** This "trust" check isn't special to deploying an app — it's just the normal, standard way SSH behaves the first time any two machines connect to each other at all; it just happens to be your production server in this case.

- `ssh $EC2_USER@$EC2_HOST "..."` — connects to your production EC2 server, and everything inside the quotes runs **on that server**, not on the Jenkins machine. This is the actual deployment step. Inside it:
  - `docker rm -f $DOCKER_CONTAINER || true` — deletes the currently running container (the old version of the app), if there is one, so it can be replaced. `|| true` means "if this fails (e.g. there was nothing to delete), don't treat it as an error — just carry on."
  - `docker pull $DOCKER_IMAGE:$DOCKER_TAG` — downloads the exact image that was just built and uploaded earlier in the pipeline (since `DOCKER_TAG` is the build number, this always grabs the newest one).
  - `docker run -d --name $DOCKER_CONTAINER -p $APP_PORT --restart unless-stopped $DOCKER_IMAGE:$DOCKER_TAG` — starts the new version:
    - `-d` — run it in the background.
    - `--name $DOCKER_CONTAINER` — give it a name (so the next deploy knows what to remove).
    - `-p $APP_PORT` — connects the app's port to the outside world.
    - `--restart unless-stopped` — keeps it running automatically, even through a server reboot.
  - `docker image prune -f` — deletes old, unused Docker images on the server to save disk space. `-f` skips the "are you sure?" confirmation.

**In short:** this stage logs into your real production server, removes the old running version of the app, downloads the brand-new image that was just built, and starts it up in its place — set to survive reboots, with old leftovers cleaned up along the way.

---

## 5. Plugins & Integrations

*(To be filled in as we go)*

---

## 6. Advanced Topics

### 6.1 Running a pipeline automatically for certain branches (`feature-*`, `release-*`)

**The simple idea:** so far, your Jenkins job only watches one fixed branch (like `main`). In real teams, people create lots of branches — `feature-login`, `feature-payment`, `release-1.0`, etc. You want: "any time someone pushes to a branch starting with `feature-` or `release-`, automatically run the pipeline for that branch too" — without manually creating a new Jenkins job every time someone makes a branch.

**The Jenkins feature for this: a "Multibranch Pipeline"**

- A normal **Pipeline** job = watches exactly one fixed branch, forever.
- A **Multibranch Pipeline** job = keeps looking at your whole repo, and automatically creates a mini-pipeline for every branch that matches a pattern you give it — with no extra clicking needed from you.

So if you push a new branch called `feature-login`, Jenkins notices it by itself, sees it matches your pattern, and starts running its pipeline automatically.

**How to set it up:**
1. Jenkins homepage → **New Item**
2. Give it a name, choose **Multibranch Pipeline**, click OK
3. Under **Branch Sources**, click **Add source → GitHub** (or Git), and give your repo URL + credentials
4. Below that, under **Behaviours**, add **"Filter by name (with wildcards)"**, and type:
   ```
   feature-* release-*
   ```
   (space-separated — meaning "match anything starting with `feature-` OR `release-`")
5. Save.

Jenkins will scan the repo, find any branch matching those patterns, and automatically build/run a pipeline for each one, using the Jenkinsfile on that branch.

### 6.2 Deploying to different environments based on Git tags (`dev-*`, `sit-*`, `sat-*`, `prd-*`)

**What is a "tag," in simple terms?** A tag is a label you stick onto one specific point in your code's history — like a sticky note on a page saying "this exact version is final." Unlike a branch (which keeps moving as new commits are added), a tag stays fixed. Teams often tag a specific commit as `dev-1.0` or `prd-2.3` to mark "this is the exact version we're deploying."

**The goal:** if someone adds a tag starting with `dev-`, deploy to the Dev server. `sit-` → SIT. `sat-` → UAT. `prd-` → Production. One pipeline, but it picks the destination based on the tag's name.

**Step 1 — make Jenkins notice tags at all.** By default, a Multibranch Pipeline only looks at branches, not tags:
1. Open the Multibranch Pipeline job → **Configure**
2. Under **Behaviours**, click **Add → Discover tags**
3. Save

Now Jenkins reacts to new tags being pushed, the same way it reacts to new branches.

**Step 2 — Jenkins tells your Jenkinsfile which tag triggered the build.** Whenever a build starts because of a tag, Jenkins automatically provides a value called `env.TAG_NAME`, holding the exact tag name (e.g. `dev-1.0`).

**Step 3 — use that tag name to pick the right environment**, inside your deploy stage:

```groovy
stage('Deploy') {
    steps {
        script {
            if (env.TAG_NAME?.startsWith('dev-')) {
                echo "Deploying to DEV environment"
                // dev deploy commands here
            } else if (env.TAG_NAME?.startsWith('sit-')) {
                echo "Deploying to SIT environment"
                // sit deploy commands here
            } else if (env.TAG_NAME?.startsWith('sat-')) {
                echo "Deploying to UAT environment"
                // uat deploy commands here
            } else if (env.TAG_NAME?.startsWith('prd-')) {
                echo "Deploying to PRODUCTION environment"
                // production deploy commands here
            } else {
                echo "This tag doesn't match any known environment — skipping deploy"
            }
        }
    }
}
```

In plain words: *look at the tag's name — if it starts with `dev-`, run the dev deploy steps; if it starts with `sit-`, run the sit ones, and so on.* Each branch of this `if`/`else if` would contain the same kind of `sshagent` + `ssh` deploy commands explained in section 4.10 — just pointed at a different server/environment each time.

**Putting both ideas together, simply:**
- **Branches** (`feature-*`, `release-*`) → Jenkins automatically **builds/tests** code as people work on it, one pipeline per branch.
- **Tags** (`dev-*`, `sit-*`, `sat-*`, `prd-*`) → Jenkins automatically **deploys** a specific, finished version to the right environment, based on what the tag is named.

---

## 7. Troubleshooting & Tips

### 7.1 "Unable to find Jenkinsfile from git..."

**What's happening:** Jenkins goes into your GitHub repo looking for a file called `Jenkinsfile`, but can't find it in the exact place it's looking.

**Common reasons:**
- Jenkins is checking the wrong branch (e.g. it's looking at `main`, but your code is actually on `master`, or the other way round).
- The file isn't where you told Jenkins to look. If your Jenkinsfile is inside a folder (like `app/Jenkinsfile`), but Jenkins was only told to look for plain `Jenkinsfile` at the very top, it won't find it.
- The file name has to be exactly `Jenkinsfile` (capital J).

**How to fix it:**
1. Open your job in Jenkins → click **Configure**.
2. Scroll to the **Pipeline** section.
3. Check **Branch Specifier** — make sure it matches your real branch name on GitHub.
4. Check **Script Path** — make sure it points exactly to where the Jenkinsfile really is (e.g. `app/Jenkinsfile` if it's inside a folder called `app`).
5. Save, then run the build again.

### 7.2 "chmod: cannot access './mvnw': No such file or directory"

**What's happening:** Picture Jenkins "standing" in a folder while it runs your commands. If your Jenkinsfile lives inside a subfolder (say, `app/`), Jenkins can find and read that file just fine — but when it starts *running* commands like `./mvnw`, it's still standing at the very top of your repo, not inside the `app/` folder where the `mvnw` file actually is. So it looks in the wrong spot and says it can't find it.

**Why this happens:** Your Jenkinsfile is at `app/Jenkinsfile`, but the commands written inside it don't know they need to first "step into" the `app/` folder before running.

**How to fix it:** Tell Jenkins to move into the `app/` folder before running your commands, using a `dir('app') { ... }` wrapper. It's like telling Jenkins "go into the app folder first, then run this."

Before:
```groovy
stage("Test"){
    steps{
        sh '''
        chmod +x ./mvnw
        ./mvnw test
        '''
    }
}
```

After (fixed):
```groovy
stage("Test"){
    steps{
        dir('app') {
            sh '''
            chmod +x ./mvnw
            ./mvnw test
            '''
        }
    }
}
```

Do the same wrapping for every stage that runs a command inside your project — `Test`, `Build`, `Docker Build`, and `Docker Push`. Basically: wherever you see `sh '''...'''` inside a stage, wrap it with `dir('app') { }` so Jenkins knows to go to the right folder first.

Once you've updated the Jenkinsfile and pushed it to GitHub, run the build again (or let the webhook start it automatically).

---

## 8. Reference Files (Full Source)

Full, unchanged copies of every file shared so far — kept here for easy copy-pasting.

### 8.1 Dockerfile (Jenkins server image)

```dockerfile
FROM jenkins/jenkins:lts-jdk21
USER root
# --------------------------------------------------
# Basic tools
# --------------------------------------------------
RUN apt-get update && \
    apt-get install -y \
        ca-certificates \
        curl \
        wget \
        git \
        unzip \
        zip \
        openssh-client \
        lsb-release \
        gnupg \
        apt-transport-https \
        jq \
        vim \
        nano \
        less \
        procps \
        iputils-ping \
        net-tools \
        && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# --------------------------------------------------
# Install Docker CLI
# --------------------------------------------------
RUN install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg \
        -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo \
      "deb [arch=$(dpkg --print-architecture) \
      signed-by=/etc/apt/keyrings/docker.asc] \
      https://download.docker.com/linux/debian \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
      > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y docker-ce-cli docker-compose-plugin && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# --------------------------------------------------
# Install Maven
# --------------------------------------------------
RUN apt-get update && \
    apt-get install -y maven && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# --------------------------------------------------
# Install kubectl
# --------------------------------------------------
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && \
    rm kubectl
# --------------------------------------------------
# Jenkins plugins
# --------------------------------------------------
USER jenkins
RUN jenkins-plugin-cli --plugins \
    workflow-aggregator \
    git \
    github \
    credentials-binding \
    ssh-agent \
    docker-workflow \
    junit \
    pipeline-stage-view \
    timestamper
```

### 8.2 docker-compose.yml (Jenkins server)

```yaml
services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: jenkins
    restart: unless-stopped
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      # Jenkins persistent data
      - jenkins_home:/var/jenkins_home
      # Give Jenkins access to host Docker
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      JAVA_OPTS: >
        -Djenkins.install.runSetupWizard=true
    networks:
      - jenkins-network
volumes:
  jenkins_home:
networks:
  jenkins-network:
    driver: bridge
```

### 8.3 Jenkinsfile (todo-backend CI pipeline)

```groovy
pipeline {
agent any 
environment{
    PROJECT_NAME="todo-backend"
    DOCKER_IMAGE="batchlcwd/simple-todo-backend"
    DOCKER_TAG="${BUILD_NUMBER}"
    EC2_HOST="13.204.45.50"
    EC2_USER="ubuntu"
    DOCKER_CONTAINER="todo-backend"
    APP_PORT="8082:8080"
}
stages{
stage("Checkout"){
        steps{
            checkout scm 
            echo "checkout successful"
            echo "Testing from github..."
        }
}
stage("Test"){
steps{
    sh '''
    chmod +x ./mvnw
    ./mvnw test
    '''
}
}
stage("Build"){
steps{
    sh '''
    
    ./mvnw clean package -DskipTests
    '''
}
}
stage("Docker Build"){
steps{
    sh '''
   
    docker build \
    -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
    
    
    '''
}
}
stage("Docker Push"){
steps{
 
    withCredentials([
        usernamePassword(
            credentialsId: 'dockerhub-credentials',
            usernameVariable: 'DOCKERHUB_USERNAME',
            passwordVariable: 'DOCKERHUB_PASSWORD'
        )
    ])
    {
        sh '''      
            echo "$DOCKERHUB_PASSWORD" | docker login \
            -u $DOCKERHUB_USERNAME \
            --password-stdin
            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
            docker image prune -af 
            
            docker images
        '''
    }
}
}
stage("Production Approval "){
steps{
    input message: 'Deploy to production?'
}
}
stage("EC2 Deploy- production server")
{
steps{
    sshagent([
        'ec2-instance-key'
    ]){
          sh '''
            mkdir -p ~/.ssh
            chmod 700 ~/.ssh
            ssh-keyscan -H "$EC2_HOST" >> ~/.ssh/known_hosts
            ssh $EC2_USER@$EC2_HOST "
                docker rm -f $DOCKER_CONTAINER || true
                docker pull $DOCKER_IMAGE:$DOCKER_TAG
                docker run -d \
                    --name $DOCKER_CONTAINER \
                    -p $APP_PORT \
                    --restart unless-stopped \
                    $DOCKER_IMAGE:$DOCKER_TAG
                docker image prune -f
            "
'''
    }
}
}
}
}
```