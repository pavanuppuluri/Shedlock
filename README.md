# Shedlock Example and Explanation

**Shedlock** is a library designed to ensure that a scheduled task (such as a background job) is executed only once in a distributed environment, where multiple nodes (or instances) might try to run the same task at the same time. It is typically used in microservices architectures or cloud-based applications to avoid race conditions and duplicate task executions.

Here’s how **Shedlock** works:

1. **Distributed Locks**: Shedlock uses a distributed locking mechanism to guarantee that only one node can execute a task at any given time.
2. **Database-backed**: It typically uses a relational database like MySQL, PostgreSQL, or even MongoDB to store lock information, ensuring that nodes can share the state of the lock.
3. **Task Execution**: Before executing a task, each node attempts to acquire the lock. If the lock is already held by another node, the task is skipped. If the lock is available, the task is executed, and the lock is released once the task is completed.

### How Shedlock Works:

1. **Job Scheduling**: Jobs (tasks) are scheduled periodically (e.g., every 5 minutes, hourly, daily).
2. **Lock Acquisition**: Each scheduled job will first attempt to acquire a lock (often stored in a database table) that identifies the job's execution state.
3. **Execution**: If the lock is available (i.e., no other node is running the job), the job executes.
4. **Releasing the Lock**: After completion, the lock is released, allowing other nodes to acquire it and execute the job if necessary.

---

### Example of Shedlock in a Java Spring Application

Below is an example of using **Shedlock** in a Java Spring Boot application.

#### 1. **Add Dependency**

In your `pom.xml` (for Maven), add the following dependency for **Shedlock** and **Spring** integration:

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>4.0.0</version> <!-- Use the latest version -->
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId> <!-- Use your database -->
    <scope>runtime</scope>
</dependency>
```

#### 2. Configure the Database Table for Locks

Shedlock stores the lock information in a table. Here’s an example SQL schema for H2 (it can be adapted for other databases):

```sql
CREATE TABLE shedlock (
    name VARCHAR(64) PRIMARY KEY,
    lock_until TIMESTAMP(3),
    locked_at TIMESTAMP(3),
    locked_by VARCHAR(255)
);
```

**Field Descriptions:**
- name: The name of the job being locked.
- lock_until: The time until which the lock is valid.
- locked_at: The timestamp when the lock was acquired.
- locked_by: The identifier of the node (e.g., hostname, IP, or UUID)

#### 3. Enable Shedlock in Spring Boot

In your `@SpringBootApplication` class, enable scheduling and configure Shedlock:

```java
import net.javacrumbs.shedlock.spring.annotation.EnableSchedulerLock;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableSchedulerLock(defaultLockAtMostFor = "PT30M")  // Set default max lock time (e.g., 30 minutes)
public class ShedlockApplication {

    public static void main(String[] args) {
        SpringApplication.run(ShedlockApplication.class, args);
    }
}
```

#### 4. Create a Scheduled Task with Shedlock

Now, you can create a scheduled task, and you will wrap it with Shedlock's lock using `@SchedulerLock`.

```java
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class ScheduledTask {

    @Scheduled(cron = "0 0/5 * * * ?")  // Run every 5 minutes
    @SchedulerLock(name = "taskName", lockAtMostFor = "PT4M", lockAtLeastFor = "PT2M")
    public void executeTask() {
        // Your scheduled task logic here
        System.out.println("Task executed at: " + System.currentTimeMillis());
    }
}
```

**Explanation:**
- @Scheduled(cron = "0 0/5 * * * ?"): This Spring annotation schedules the task to run every 5 minutes.
- @SchedulerLock(name = "taskName"): This is where you define the lock. The name is used to identify the job in the database.
- lockAtMostFor: Defines the maximum duration the lock can be held (e.g., 4 minutes).
- lockAtLeastFor: Defines the minimum duration the lock should be held (e.g., 2 minutes). This ensures the task doesn't start before this time.

#### 5. Database Configuration

Make sure your database (e.g., H2, MySQL, PostgreSQL) is correctly configured to store the lock information. Here’s an example of an `application.properties` for H2:

```properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=update
```

#### 6. How It Works

- The scheduled task runs at the defined interval (every 5 minutes in this case).
- When the task starts, it tries to acquire a lock from the database. If another instance of the application has already acquired the lock (and hasn't finished), it will skip the task.
- Once the task completes, the lock is released, allowing other nodes to acquire the lock and execute the task if needed.


#### 7. Example Use Case

Imagine you have a task that needs to be run periodically, such as syncing data with an external service. In a distributed system, you don't want multiple nodes trying to run the task at the same time, as this could lead to data duplication or conflicts.

Shedlock ensures that only one instance of your application runs the task at any given time, preventing concurrent executions and ensuring consistency in your task execution.

#### Conclusion

- **Shedlock** is essential for preventing multiple executions of the same scheduled task in a distributed environment.
- It provides a simple mechanism based on **distributed locks** that prevents race conditions.
- By using **database-backed locks**, you can ensure that the task execution is synchronized across multiple instances or nodes.









