# Getting Started With Spring On OpenShift

## Deploying a Spring Boot application to OpenShift

```
# Preparation work:
$ oc login -u developer -p developer
$ oc new-project dev --display-name="Dev - Spring Boot App"

# Create PostgreSQL database:
$ oc new-app -e POSTGRESQL-USER=luke \
    -e POSTGRESQL_PASSWORD=secret \
    -e POSTGRESQL_DATABASE=my_data \
    openshift/postgresql-92-centos7 \
    --name=my-database

# Matching Deployment manifest in application's 'src/main/fabric8/deployment.yaml' file:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${project.artifactId}
spec:
  template:
    spec:
      containers:
      - env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: my-database-secret
              key: user
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-database-secret
              key: password
        - name: JAVA_OPTIONS
          value: "-Dspring.profiles.active=openshift"

# Corresponding Secret manifest:
apiVersion: v1
kind: Secret
metadata:
  name: my-database-secret
stringData:
  user: luke
  password: secret

# Route manifest (OpenShift's variant of Kubernetes Ingress):
apiVersion: v1
kind: Route
metadata:
  name: ${project.artifactId}
spec:
  port:
    targetPort: 8080
  to:
    kind: Service
    name: ${project.artifactId}

# 'application.properties' makes use of environment variables provided in the Deployment manifest:
spring.datasource.url=jdbc:postgresql://${MY_DATABASE_SERVICE_HOST}:${MY_DATABASE_SERVICE_PORT}/my_data
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.hibernate.ddl-auto=create

# Invocation of fabric8 plugin in POM:
# ...
<profiles>
  <!-- ... -->
  <profile>
    <id>openshift</id>
    <dependencies>
      <!-- ... -->
    </dependencies>
    <build>
      <plugins>
        <plugin>
          <groupId>io.fabric8</groupId>
          <artifactId>fabric8-maven-plugin</artifactId>
          <version>${fabric8-maven-plugin.version}</version>
          <executions>
            <execution>
              <id>fmp</id>
              <goals>
                <goal>resource</goal>
                <goal>build</goal>
              </goals>
            </execution>
          </executions>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
# ...

# Deploy application:
mvn package fabric8:deploy -Popenshift -DskipTests

# Check rollout status:
$ oc rollout status dc/fruits

```
