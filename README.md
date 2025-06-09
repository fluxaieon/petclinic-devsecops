# Spring PetClinic CI/CD Pipeline

This repository contains the Spring PetClinic application, enhanced with a GitHub Actions CI/CD pipeline. The pipeline automates the following steps:

1.  **Code Compilation:** Compiles the Java source code using Maven.
2.  **Test Execution:** Runs the unit and integration tests.
3.  **Docker Image Packaging:** Packages the application as a runnable Docker image.
4.  **Image Publishing to JFrog Artifactory:** Pushes the built Docker image to a specified JFrog Artifactory repository.
5.  **Dependency Resolution from JFrog Artifactory:** All Maven dependencies are resolved through a JFrog Artifactory virtual repository (`maven-virtual`) which proxies Maven Central.
6.  **Base Image Resolution from JFrog Artifactory:** The Docker base image (`openjdk:17-jdk-slim`) is pulled through a JFrog Artifactory virtual Docker repository (`docker-virtual`) which proxies Docker Hub.
7.  **XRay Scan (Optional):** Triggers a security scan of the Docker image using JFrog XRay (if enabled and configured).

## **Project Structure**

* `src/`: Spring PetClinic source code.
* `pom.xml`: Maven project object model.
* `Dockerfile`: Defines how to build the Docker image for the application.
* `.github/workflows/ci-cd.yml`: The GitHub Actions workflow definition.

## **Pipeline Details**

The pipeline is triggered on `push` and `pull_request` events to the `main` branch.

### **Key Steps in `ci-cd.yml`:**

* **Checkout code:** Fetches the repository content.
* **Set up JDK 17:** Configures the Java Development Kit environment.
* **Install & Configure JFrog CLI:** Downloads and sets up the JFrog CLI (`jf` command) for interacting with Artifactory. Maven and Docker configurations are set up to use your JFrog instance.
* **Configure Maven for JFrog Artifactory:** Overrides the default Maven `settings.xml` to direct all dependency resolution to your JFrog Artifactory virtual Maven repository (`maven-virtual`).
* **Compile the code:** Executes `mvn clean compile`.
* **Run the tests:** Executes `mvn test`.
* **Package the project (JAR):** Creates the runnable JAR file using `mvn package -DskipTests`.
* **Build Docker image:** Constructs the Docker image based on the `Dockerfile`. The base image is pulled via your JFrog Artifactory virtual Docker repository (`docker-virtual`).
* **Publish Docker image to JFrog Artifactory:** Pushes the newly built Docker image to your specified JFrog Artifactory local Docker repository (`docker-hub-local`).
* **Trigger JFrog XRay Scan (Optional):** Initiates an XRay scan on the published Docker image.
* **Get XRay Scan Data (Optional):** Attempts to retrieve the XRay scan results and save them as an artifact.

## **JFrog Configuration Assumed**

This pipeline assumes you have a JFrog Platform instance with the following repositories configured (matching the names used in `ci-cd.yml`):

* **Maven Repositories:**
    * `maven-central-remote` (Remote, pointing to `https://repo.maven.apache.org/maven2/`)
    * `maven-local` (Local)
    * `maven-virtual` (Virtual, aggregating `maven-central-remote` and `maven-local`)
* **Docker Repositories:**
    * `docker-hub-remote` (Remote, pointing to `https://registry-1.docker.io/`)
    * `docker-hub-local` (Local)
    * `docker-virtual` (Virtual, aggregating `docker-hub-remote` and `docker-hub-local`)

## **How to Run the Project (Docker Image)**

To obtain and run the built Docker image from your JFrog Artifactory:

1.  **Login to your JFrog Artifactory Docker registry:**
    ```bash
    docker login YOUR_JFROG_PLATFORM_URL_WITHOUT_HTTPS_OR_HTTP/artifactory/docker-virtual -u YOUR_JFROG_USERNAME -p YOUR_JFROG_ACCESS_TOKEN
    # Example: docker login mycompany.jfrog.io/artifactory/docker-virtual -u your_user -p <your_access_token>
    ```
    *Replace `YOUR_JFROG_PLATFORM_URL_WITHOUT_HTTPS_OR_HTTP` with your actual JFrog domain (e.g., `mycompany.jfrog.io`). `docker-virtual` is used here for logging in as it's the virtual repo that aggregates everything.*

2.  **Pull the Docker image:**
    ```bash
    docker pull YOUR_JFROG_PLATFORM_URL_WITHOUT_HTTPS_OR_HTTP/artifactory/docker-hub-local/spring-petclinic:COMMIT_SHA
    # Example: docker pull mycompany.jfrog.io/artifactory/docker-hub-local/spring-petclinic:a1b2c3d4e5f67890abcdef1234567890
    ```
    *Replace `YOUR_JFROG_PLATFORM_URL_WITHOUT_HTTPS_OR_HTTP` with your actual JFrog domain. Replace `COMMIT_SHA` with the specific commit SHA from your GitHub Actions workflow run (you can find this in the workflow run details).*

3.  **Run the Docker image:**
    ```bash
    docker run -p 8080:8080 YOUR_JFROG_PLATFORM_URL_WITHOUT_HTTPS_OR_HTTP/artifactory/docker-hub-local/spring-petclinic:COMMIT_SHA
    ```

    The Spring PetClinic application will be accessible at `http://localhost:8080` in your web browser.

## **XRay Scan Data Export**

## XRay Scan Data Export

The `xray_scan_data.json` file, containing the XRay scan results for the built Docker image, will be available as an artifact in your GitHub Actions workflow run.

**Additionally, a manually obtained XRay Scan Report is available as a JSON file directly in the repository under the `/scans/` folder.**

To retrieve it from the pipeline artifacts:
1.  Go to your GitHub repository -> `Actions` tab.
2.  Click on the latest successful workflow run.
3.  Scroll down to the "Artifacts" section.
4.  Download the `xray-scan-data` artifact. This ZIP file will contain `xray_scan_data.json`.

**Note:** Due to the asynchronous nature of XRay scans and potential API timing issues (as experienced during development), the `xray_scan_data.json` artifact downloaded from the pipeline might sometimes still contain a `404 Not Found` error, even if the scan completed and is visible in the JFrog UI. For comprehensive and definitive scan results, always refer directly to the XRay tab for your artifact in the JFrog Platform UI.
