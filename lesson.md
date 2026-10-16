# Lesson 4.11: CI Practicals - Applying CI to Simple-CRM-Lite

## Learning Objectives

By the end of this lesson, you will be able to:
1. **Set up** a Continuous Integration pipeline for a Spring Boot application with database
2. **Create** CircleCI workflows with build, test, and publish jobs
3. **Configure** PostgreSQL service in CircleCI for running tests
4. **Push** Docker images to Docker Hub automatically from a CI pipeline
5. **Deploy** the CI-built image and a PostgreSQL database to Railway on your own (challenge)

---

## Prerequisites

Before starting this coaching session, ensure you have:

### ✅ Completed Previous Lessons
- [ ] Lesson 4.7: Continuous Integration (CircleCI basics)
- [ ] Lesson 4.8: Containerized simple-crm-lite application
- [ ] Lesson 4.10: Deployed to Railway from Docker Hub, with PostgreSQL (for the Part 6 challenge)

### ✅ Accounts Ready
- [ ] GitHub account (free)
- [ ] Docker Hub account (free) - from Lesson 4.5
- [ ] CircleCI account (free) - from Lesson 4.7

### ✅ Project Ready
- [ ] simple-crm-lite project on your computer (from Lesson 4.8)
- [ ] Dockerfile exists in project root
- [ ] docker-compose.yml exists in project root
- [ ] Application runs locally: `mvn spring-boot:run`

### ✅ Tools Installed
- [ ] Git installed and configured
- [ ] Java 21
- [ ] Maven
- [ ] Docker Desktop running

---

## Introduction

In Lesson 4.7, you learned Continuous Integration using CircleCI with the devops-demo project. In Lesson 4.8, you containerized the simple-crm-lite application.

**Today, you'll combine both skills:**
- Set up a CI pipeline for simple-crm-lite
- Automatically build, test, and publish Docker images
- Use the SAME approach as Lesson 4.7, but for your simple-crm-lite project

**Why This Matters:**
- Every code push triggers automatic testing
- Bugs are caught immediately
- Docker images are built automatically
- Ready for deployment - you'll deploy it to Railway at the end of this lesson

---

## Part 1 - Setup and Review

### Step 1: Review CI Concepts

**Quick Review from Lesson 4.7:**

**What is Continuous Integration (CI)?**
- Automatically build and test code when you push to GitHub
- Catch errors early
- Ensure code always works

**CircleCI Pipeline:**
1. **Build Job** - Compile code, create JAR file
2. **Test Job** - Run tests (with database)
3. **Publish Job** - Build Docker image, push to Docker Hub

**Today:** Apply this exact pattern to simple-crm-lite!

---

### Step 2: Check Your Accounts

**Instructor will verify everyone has:**

1. **GitHub Account**
   - Go to [https://github.com](https://github.com)
   - You should be logged in

2. **Docker Hub Account**
   - Go to [https://hub.docker.com](https://hub.docker.com)
   - You should be logged in
   - Remember your username (you'll need it!)

3. **CircleCI Account**
   - Go to [https://circleci.com](https://circleci.com)
   - Login with GitHub (you did this in Lesson 4.7)
   - Should see CircleCI dashboard

---

### Step 3: Create GitHub Repository

You need to push your simple-crm-lite project to GitHub.

**3.1: Create New Repository on GitHub**

1. Go to [https://github.com/new](https://github.com/new)

2. Fill in details:
   - **Repository name:** `simple-crm-lite`
   - **Description:** `Simple CRM application for learning CI/CD`
   - **Public** (recommended) or Private
   - **DO NOT** initialize with README (you already have a project)

3. Click **"Create repository"**

**3.2: Push Your Code to GitHub**

Open terminal in your `simple-crm-lite` project folder and run:

```bash
# Initialize git (if not already done)
git init

# Add all files
git add .

# Commit
git commit -m "Initial commit - simple-crm-lite with Docker"

# Add GitHub remote (replace YOUR-USERNAME with your GitHub username)
git remote add origin https://github.com/YOUR-USERNAME/simple-crm-lite.git

# Push to GitHub
git branch -M main
git push -u origin main
```

**Verify:** Go to your GitHub repository in browser - you should see all your files!

---

### Step 4: Review Project Structure

Let's quickly review what we have from Lesson 4.8:

**Instructor will show on screen:**

```
simple-crm-lite/
├── src/
│   ├── main/
│   │   ├── java/com/example/simplecrmlite/
│   │   │   ├── SimpleCrmLiteApplication.java
│   │   │   ├── controller/
│   │   │   │   └── CustomerController.java
│   │   │   ├── repository/
│   │   │   │   └── CustomerRepository.java
│   │   │   └── model/
│   │   │       └── Customer.java
│   │   └── resources/
│   │       └── application.properties
│   └── test/
│       └── java/...
├── Dockerfile                    ← From Lesson 4.8
├── docker-compose.yml           ← From Lesson 4.8
└── pom.xml
```

**Key Points:**
- ✅ We have a working Spring Boot app
- ✅ We have Dockerfile
- ✅ We have docker-compose.yml
- ✅ Controller talks directly to `CustomerRepository` (no service layer — kept simple, as in Lesson 4.8)
- ❓ We need to add tests for CI

---

## Part 2 - Add a Simple Test

For CI to be meaningful, we need tests! Let's add ONE simple test.

### Step 1: Create Test Directory Structure

Check if this directory exists:
```
src/test/java/com/example/simplecrmlite/controller/
```

If it doesn't exist, create it:

```bash
mkdir -p src/test/java/com/example/simplecrmlite/controller
```

---

### Step 2: Add Test Dependency in pom.xml

Open `pom.xml` and verify you have the test dependencies. simple-crm-lite uses **Spring Boot 4**, which has a test starter for each main starter:

```xml
<dependencies>
    <!-- Other dependencies... -->

    <!-- Spring Boot 4 test dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webmvc-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**Note:** These should already be there from when the project was created. If not, add them!

---

### Step 3: Create a Simple Controller Test

Create a new file:
```
src/test/java/com/example/simplecrmlite/controller/CustomerControllerTest.java
```

**Copy this test code:**

```java
package com.example.simplecrmlite.controller;

import com.example.simplecrmlite.model.Customer;
import com.example.simplecrmlite.repository.CustomerRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import java.util.Arrays;
import java.util.List;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * Test class for CustomerController
 * Tests the GET /customers endpoint
 */
@WebMvcTest(CustomerController.class)
public class CustomerControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private CustomerRepository customerRepository;

    /**
     * Test: GET /customers should return list of customers
     */
    @Test
    public void testGetAllCustomers() throws Exception {
        // Arrange - Create test data
        Customer customer1 = new Customer();
        customer1.setId(1L);
        customer1.setFirstName("John");
        customer1.setLastName("Doe");
        customer1.setEmail("john@example.com");

        Customer customer2 = new Customer();
        customer2.setId(2L);
        customer2.setFirstName("Jane");
        customer2.setLastName("Smith");
        customer2.setEmail("jane@example.com");

        List<Customer> customers = Arrays.asList(customer1, customer2);

        // Mock the repository to return test data
        when(customerRepository.findAll()).thenReturn(customers);

        // Act & Assert - Make request and verify response
        mockMvc.perform(get("/customers")
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].firstName").value("John"))
                .andExpect(jsonPath("$[0].lastName").value("Doe"))
                .andExpect(jsonPath("$[1].firstName").value("Jane"))
                .andExpect(jsonPath("$[1].lastName").value("Smith"));
    }
}
```

**Understanding the Test:**

1. **`@WebMvcTest`** - Tests only the web layer (controller)
2. **`@MockitoBean`** - Creates a fake `CustomerRepository` (we don't need a real database for this test — matches how `CustomerController` calls `CustomerRepository` directly, same as in Lesson 4.8)
3. **`mockMvc.perform(get(...))`** - Simulates HTTP GET request
4. **`andExpect(...)`** - Verifies the response

**This is a UNIT test - it doesn't need a database!**

> **Spring Boot 4 note:** `@WebMvcTest` now lives in `org.springframework.boot.webmvc.test.autoconfigure` (as in Lesson 4.7), and `@MockBean` has been removed - use `@MockitoBean` instead. Older tutorials still show the Spring Boot 3 versions.

---

### Step 4: Run the Test Locally

**Run the controller test to make sure it works:**

```bash
mvn test -Dtest=CustomerControllerTest
```

**Expected output:**
```
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

**Why `-Dtest=CustomerControllerTest`?** The project also has `SimpleCrmLiteApplicationTests`, which starts the whole application - and the application needs a PostgreSQL database. You don't have one set up for local tests, so we run only the controller test here. In CircleCI, the test job gives the app a PostgreSQL database, so **both** tests run there.

---

### Step 5: Commit and Push Test

```bash
git add .
git commit -m "Add CustomerController test"
git push origin main
```

**Verify on GitHub:** You should see the new test file.

---

## Part 3 - Create CircleCI Pipeline

Now we'll create the CI pipeline! This is similar to Lesson 4.7, but adapted for simple-crm-lite.

### Step 1: Create CircleCI Configuration File

In your project root, create this directory and file:

```bash
mkdir -p .circleci
```

Create file: `.circleci/config.yml`

**Note:** The dot (`.`) at the beginning is important!

---

### Step 2: Build Job

Add this to your `.circleci/config.yml`:

```yml
version: 2.1

jobs:
  # ==========================================
  # BUILD JOB - Compile and package the app
  # ==========================================
  build:
    docker:
      - image: cimg/openjdk:21.0
    
    steps:
      - checkout
      
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      
      - run:
          name: Install dependencies and build
          command: |
            echo "Building the application..."
            mvn clean install -DskipTests
      
      - save_cache:
          paths:
            - ~/.m2
          key: maven-deps-{{ checksum "pom.xml" }}
      
      - persist_to_workspace:
          root: .
          paths:
            - target/*.jar
```

**What This Does:**
1. Uses Java 21 Docker image (same version as your local Java)
2. Downloads your code from GitHub
3. Caches Maven dependencies (faster builds)
4. Runs `mvn clean install -DskipTests` (builds JAR, skips tests for now)
5. Saves JAR file for test job

---

### Step 3: Test Job

Add the test job to your `.circleci/config.yml` (below the build job):

```yml
  # ==========================================
  # TEST JOB - Run tests with PostgreSQL
  # ==========================================
  test:
    docker:
      - image: cimg/openjdk:21.0
      
      # Secondary container: PostgreSQL database for tests
      - image: cimg/postgres:16.0
        environment:
          POSTGRES_DB: simplecrmlite
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
    
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/simplecrmlite
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: create
    
    steps:
      - checkout
      
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      
      - run:
          name: Install PostgreSQL client
          command: |
            sudo apt-get update && sudo apt-get install -y postgresql-client
      
      - run:
          name: Wait for PostgreSQL
          command: |
            for i in {1..10}; do
              if pg_isready -h localhost -p 5432 -U postgres; then
                echo "PostgreSQL is ready!"
                exit 0
              fi
              echo "Waiting for PostgreSQL... ($i/10)"
              sleep 3
            done
            echo "PostgreSQL failed to start"
            exit 1
      
      - run:
          name: Run tests
          command: |
            echo "Running tests..."
            mvn test
      
      - store_test_results:
          path: target/surefire-reports
      
      - store_artifacts:
          path: target/surefire-reports
```

**What This Does:**
1. Starts TWO containers: Java (main) + PostgreSQL (database)
2. Sets environment variables for database connection
3. Installs the PostgreSQL client tools (`pg_isready` isn't included in the `cimg/openjdk` image by default, so this step adds it)
4. Waits for PostgreSQL to be ready
5. Runs `mvn test` (your test will run against real database)
6. Saves test results

**Why a database?** `CustomerControllerTest` doesn't need one, but `SimpleCrmLiteApplicationTests` starts the whole application, which connects to PostgreSQL. The PostgreSQL container gives it a real database in CI.

---

### Step 4: Publish Job

Add the publish job to your `.circleci/config.yml`:

```yml
  # ==========================================
  # PUBLISH JOB - Build and push Docker image
  # ==========================================
  publish:
    docker:
      - image: cimg/base:stable
    
    steps:
      - checkout
      
      - attach_workspace:
          at: .
      
      - setup_remote_docker
      
      - run:
          name: Build Docker image
          command: |
            echo "Building Docker image..."
            docker build -t $DOCKER_USERNAME/simple-crm-lite:latest .
      
      - run:
          name: Push to Docker Hub
          command: |
            echo "Logging in to Docker Hub..."
            echo $DOCKER_TOKEN | docker login -u $DOCKER_USERNAME --password-stdin
            echo "Pushing image to Docker Hub..."
            docker push $DOCKER_USERNAME/simple-crm-lite:latest
```

**What This Does:**
1. Sets up Docker environment
2. Builds Docker image using your Dockerfile
3. Logs in to Docker Hub using credentials
4. Pushes image to Docker Hub

---

### Step 5: Define Workflow

Add the workflow at the END of your `.circleci/config.yml`:

```yml
# ==========================================
# WORKFLOW - Run jobs in sequence
# ==========================================
workflows:
  build_test_publish:
    jobs:
      - build
      - test:
          requires:
            - build
      - publish:
          context: docker-hub
          requires:
            - test
```

---

### Complete .circleci/config.yml

Here's your complete file for reference:

```yml
version: 2.1

jobs:
  build:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Install dependencies and build
          command: |
            echo "Building the application..."
            mvn clean install -DskipTests
      - save_cache:
          paths:
            - ~/.m2
          key: maven-deps-{{ checksum "pom.xml" }}
      - persist_to_workspace:
          root: .
          paths:
            - target/*.jar

  test:
    docker:
      - image: cimg/openjdk:21.0
      - image: cimg/postgres:16.0
        environment:
          POSTGRES_DB: simplecrmlite
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/simplecrmlite
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: create
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Install PostgreSQL client
          command: |
            sudo apt-get update && sudo apt-get install -y postgresql-client
      - run:
          name: Wait for PostgreSQL
          command: |
            for i in {1..10}; do
              if pg_isready -h localhost -p 5432 -U postgres; then
                echo "PostgreSQL is ready!"
                exit 0
              fi
              echo "Waiting for PostgreSQL... ($i/10)"
              sleep 3
            done
            echo "PostgreSQL failed to start"
            exit 1
      - run:
          name: Run tests
          command: |
            echo "Running tests..."
            mvn test
      - store_test_results:
          path: target/surefire-reports
      - store_artifacts:
          path: target/surefire-reports

  publish:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - attach_workspace:
          at: .
      - setup_remote_docker
      - run:
          name: Build Docker image
          command: |
            echo "Building Docker image..."
            docker build -t $DOCKER_USERNAME/simple-crm-lite:latest .
      - run:
          name: Push to Docker Hub
          command: |
            echo "Logging in to Docker Hub..."
            echo $DOCKER_TOKEN | docker login -u $DOCKER_USERNAME --password-stdin
            echo "Pushing image to Docker Hub..."
            docker push $DOCKER_USERNAME/simple-crm-lite:latest

workflows:
  build_test_publish:
    jobs:
      - build
      - test:
          requires:
            - build
      - publish:
          context: docker-hub
          requires:
            - test
```

---

## Part 4 - Setup and Run Pipeline

### Step 1: Commit CircleCI Configuration

```bash
git add .circleci/config.yml
git commit -m "Add CircleCI configuration"
git push origin main
```

**Verify on GitHub:** You should see `.circleci/config.yml` in your repository.

---

### Step 2: Connect CircleCI to GitHub Repository

**2.1: Go to CircleCI**

1. Open [https://app.circleci.com](https://app.circleci.com)
2. Login with GitHub

**2.2: Add Project**

1. Click **"Projects"** in left sidebar
2. Find your `simple-crm-lite` repository
3. Click **"Set Up Project"**

**2.3: Configure Project**

1. CircleCI will detect your `.circleci/config.yml`
2. Select **"Use Existing Config"**
3. Click **"Start Building"**

**CircleCI starts running the pipeline.** If your `docker-hub` context from Lesson 4.7 is set up, all three jobs can pass. If the publish job fails, check your Docker Hub credentials in Step 3.

---

### Step 3: Check Your Docker Hub Credentials

The publish job uses the **`docker-hub` context** you created in Lesson 4.7 (`context: docker-hub` in the workflow). It holds two environment variables:

| Variable | Value |
|---|---|
| `DOCKER_USERNAME` | Your Docker Hub username (e.g. `john123`) |
| `DOCKER_TOKEN` | Your Docker Hub personal access token |

Contexts belong to your CircleCI organisation, so every project can reuse them - no need to add the credentials again.

**3.1: Check the context**

1. In CircleCI, click **"Organization Settings"**
2. Click **"Contexts"** and open **`docker-hub`**
3. Check that both `DOCKER_USERNAME` and `DOCKER_TOKEN` are there

**3.2: Token expired or context missing?**

1. In Docker Hub, go to **Account settings → Personal access tokens** and generate a new token (`Read & Write`)
2. In the `docker-hub` context, add or update `DOCKER_TOKEN` with the new token (and `DOCKER_USERNAME` if missing)

**IMPORTANT:** Never put passwords or tokens in code! Always use a context or environment variables.

---

### Step 4: Trigger Pipeline

Now let's trigger the pipeline to run again!

**Option 1: Push a small change**

```bash
git add .
git commit -m "Trigger pipeline"
git push origin main
```

**Option 2: Rerun from CircleCI**

1. In CircleCI, go to your project
2. Click on the failed build
3. Click **"Rerun Workflow from Failed"**

---

### Step 5: Watch Pipeline Run

**Instructor will project on screen:**

1. **Go to CircleCI Dashboard**
2. **Click on your running pipeline**
3. **Watch each job:**
   - ⏳ build (yellow = running)
   - ⏳ test (waiting for build)
   - ⏳ publish (waiting for test)

**Timeline:**
- **Build job:** ~2-3 minutes
- **Test job:** ~2-3 minutes (includes database startup)
- **Publish job:** ~2-3 minutes (builds and pushes Docker image)

**Total:** ~7-10 minutes

**What to watch for:**

**Build Job:**
- See Maven downloading dependencies (first time only)
- See "BUILD SUCCESS"
- Job turns green ✅

**Test Job:**
- See "Waiting for PostgreSQL"
- See "PostgreSQL is ready!"
- See "Tests run: 2, Failures: 0" (the controller test and `SimpleCrmLiteApplicationTests`)
- Job turns green ✅

**Publish Job:**
- See "Building Docker image..."
- See "Pushing image to Docker Hub..."
- See "latest: digest: sha256:..."
- Job turns green ✅

**All jobs green = SUCCESS!** 🎉

---

## Part 5 - Verify and Wrap-up

### Step 1: Verify Docker Image on Docker Hub

**Check if image was published:**

1. Go to [https://hub.docker.com](https://hub.docker.com)
2. Login
3. Go to **"Repositories"**
4. You should see **`simple-crm-lite`** repository
5. Click on it
6. You should see tag **`latest`** with timestamp

**Success!** Your CI pipeline automatically built and published the Docker image!

---

### Step 2: Pull and Run the Image Locally

**Let's test the image that CI built:**

```bash
# Pull the image from Docker Hub
docker pull YOUR-DOCKERHUB-USERNAME/simple-crm-lite:latest

# Stop any running containers from previous lessons
docker compose down

# Run just the database
docker run -d \
  --name simple-crm-db \
  -e POSTGRES_DB=simplecrmlite \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=password \
  -p 5433:5432 \
  postgres:16-alpine

# Wait 5 seconds for database to start
sleep 5

# Run the app (using image from Docker Hub)
docker run -d \
  --name simple-crm-app \
  -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5433/simplecrmlite \
  -e SPRING_DATASOURCE_USERNAME=postgres \
  -e SPRING_DATASOURCE_PASSWORD=password \
  YOUR-DOCKERHUB-USERNAME/simple-crm-lite:latest

# Wait 10 seconds for app to start
sleep 10

# Test it
curl http://localhost:8080/customers
```

**You should see the customer list!** (An empty list `[]` is fine - the database is new.)

> The app needs all three database variables - `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME` and `SPRING_DATASOURCE_PASSWORD`. If any is missing, it fails to start with `Could not resolve placeholder`.

**This proves:** The Docker image built by CI works perfectly!

**Clean up:**
```bash
docker stop simple-crm-app simple-crm-db
docker rm simple-crm-app simple-crm-db
```

---

### Step 3: Understanding What You Built

**Instructor will recap:**

**You now have a complete CI pipeline that:**

1. **Automatically triggers** when you push code to GitHub
2. **Builds your application** (compiles Java, creates JAR)
3. **Runs tests** with a real PostgreSQL database
4. **Builds Docker image** using your Dockerfile
5. **Pushes to Docker Hub** automatically
6. **All in 7-10 minutes** without any manual work!

**Real-world impact:**
- Every developer's code is automatically tested
- No "it works on my machine" problems
- Docker images ready to deploy
- Bugs caught immediately

**This is how professional teams work!**

---

## Part 6 - Challenge: Deploy simple-crm-lite to Railway

Your CI pipeline has published a Docker image of simple-crm-lite to Docker Hub. **It's ready for deployment!**

Deploy it to Railway, together with a PostgreSQL database.

> **No step-by-step instructions this time.** Use **Lesson 4.10** as your reference: Part 3 (deploy to Railway from Docker Hub) and Part 4 (add a PostgreSQL database and connect it). Working it out yourself is the point of this challenge.

**Your image:**

```
YOUR-DOCKERHUB-USERNAME/simple-crm-lite:latest
```

**You're done when:**

- [ ] simple-crm-lite and a PostgreSQL database run in the same Railway project
- [ ] Your app has a public Railway URL
- [ ] `GET /customers` on your Railway URL returns JSON (an empty list `[]` at first)
- [ ] You can create a customer with `POST /customers` and see it in `GET /customers`

**Tip:** Use the same requests you used to test the API in Lesson 4.8 - just replace `http://localhost:8080` with your Railway URL.

---

## Troubleshooting Guide

### Issue 1: Build job fails - "Could not find or load main class"

**Cause:** Java version mismatch

**Solution:**
```yml
docker:
  - image: cimg/openjdk:21.0  # Must match your local Java version
```

---

### Issue 2: Test job fails - "Connection refused to PostgreSQL"

**Cause:** PostgreSQL not ready yet, or wrong connection settings

**Solution 1:** Increase wait time
```yml
for i in {1..15}; do  # Changed from 10 to 15
```

**Solution 2:** Check environment variables
```yml
environment:
  SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/simplecrmlite
```

**Solution 3:** Make sure the "Install PostgreSQL client" step ran successfully before the wait step — without it, `pg_isready` won't be found at all, which looks similar to a connection error in the logs but is actually a missing-command error.

---

### Issue 3: Publish job fails - "denied: requested access to resource is denied"

**Cause:** Wrong Docker Hub credentials or image name

**Solution:**
1. Verify the `docker-hub` context in CircleCI (Organization Settings → Contexts):
   - `DOCKER_USERNAME` = your exact Docker Hub username
   - `DOCKER_TOKEN` = a valid personal access token with `Read & Write` access
   - The workflow has `context: docker-hub` on the publish job
2. Verify image name in config:
   ```yml
   docker build -t $DOCKER_USERNAME/simple-crm-lite:latest .
   ```

---

### Issue 4: Publish job fails - "unauthorized: incorrect username or password"

**Solution:**
1. Generate a new Docker Hub personal access token (your old one may have expired)
2. In CircleCI, go to Organization Settings → Contexts → `docker-hub`
3. Update `DOCKER_TOKEN` with the new token (no spaces)
4. Rerun the workflow

---

### Issue 5: Tests pass locally but fail in CircleCI

**Solution:**
1. Check pom.xml has all dependencies
2. Run `mvn clean test` locally to verify
3. Check CircleCI logs for specific error
4. Ensure test doesn't depend on local files

---

### Issue 6: Pipeline triggers but no jobs run

**Solution:**
1. Verify file is at `.circleci/config.yml` (note the dot!)
2. Check YAML syntax (no tabs, proper indentation)
3. Commit and push config file
4. Manually trigger in CircleCI

---

### Issue 7: Build is slow (>10 minutes)

**Solution:** Ensure you have cache configured:
```yml
- restore_cache:
    keys:
      - maven-deps-{{ checksum "pom.xml" }}
- save_cache:
    paths:
      - ~/.m2
    key: maven-deps-{{ checksum "pom.xml" }}
```

---

### Issue 8: App can't connect to database when testing locally (Linux users)

**Cause:** `host.docker.internal` doesn't work on Linux

**Solution:** Use Docker network instead:
```bash
# Create a network
docker network create simple-crm-network

# Run database on the network
docker run -d \
  --name simple-crm-db \
  --network simple-crm-network \
  -e POSTGRES_DB=simplecrmlite \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=password \
  -p 5433:5432 \
  postgres:16-alpine

# Run app on the same network
docker run -d \
  --name simple-crm-app \
  --network simple-crm-network \
  -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://simple-crm-db:5432/simplecrmlite \
  -e SPRING_DATASOURCE_USERNAME=postgres \
  -e SPRING_DATASOURCE_PASSWORD=password \
  YOUR-DOCKERHUB-USERNAME/simple-crm-lite:latest

# Test
curl http://localhost:8080/customers
```

---

## Summary

### What You Accomplished Today

1. ✅ Set up GitHub repository for simple-crm-lite
2. ✅ Created a unit test for CustomerController
3. ✅ Created complete CircleCI pipeline with 3 jobs
4. ✅ Configured PostgreSQL service in CI
5. ✅ Reused your Docker Hub credentials securely with a CircleCI context
6. ✅ Ran successful CI pipeline
7. ✅ Published Docker image automatically
8. ✅ (Challenge) Deployed simple-crm-lite and PostgreSQL to Railway

### Key Takeaways

**1. CI automates quality:**
- Tests run automatically on every push
- Bugs caught immediately
- No manual testing needed

**2. Pipeline pattern is reusable:**
- Same structure for any Spring Boot app
- Just change app name and credentials
- Applies to your other projects

**3. Docker + CI = Deployment Ready:**
- Every successful build creates a Docker image
- Image is on Docker Hub, ready to deploy
- You deployed it to Railway yourself, using Lesson 4.10 as your guide

**4. Professional workflow:**
- This is how real companies work
- You now have industry-standard CI setup
- Employers value this skill

---

## Additional Resources

- [CircleCI Documentation](https://circleci.com/docs/)
- [CircleCI Java/Maven Guide](https://circleci.com/docs/language-java-maven/)
- [CircleCI PostgreSQL Service](https://circleci.com/docs/postgres-config/)
- [CircleCI Status](https://status.circleci.com/)
- [Docker Hub](https://hub.docker.com)
- [CircleCI Community](https://discuss.circleci.com/)