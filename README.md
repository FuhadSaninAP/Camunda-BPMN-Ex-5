# Student Leave Workflow — Camunda 8 SaaS + Spring Boot

A Student Leave Application process that runs on a **Camunda 8 SaaS** cluster, with all business logic in a **Spring Boot** app on your laptop. It demonstrates the four things every Camunda 8 + Spring project needs:

1. Deploying models from code (`@Deployment`)
2. Starting a process instance over REST (`POST /leave`)
3. Running service tasks as job workers (`@JobWorker`)
4. A human task with a form in Tasklist

The app only makes **outbound** connections to the cluster (the controller starts instances, workers poll for jobs), so it works from campus or home networks with no port forwarding.

## Process overview

1. A student submits a leave request via `POST /leave`.
2. **Validate leave request** (`validate-leave` worker) computes the number of days and checks the dates are not in the past.
3. **Decide approval route** (DMN `leave-routing`) returns a route, and a gateway follows it:

   | valid | leaveDays | route          | What happens                              |
   |-------|-----------|----------------|-------------------------------------------|
   | false | any       | `REJECT`       | Skips approval, student is told REJECTED  |
   | true  | ≤ 2       | `AUTO_APPROVE` | Approved automatically                    |
   | true  | > 2       | `FA_APPROVAL`  | Faculty Advisor approves in Tasklist      |

4. **Notify student** (`notify-student` worker) logs the final decision (stand-in for an e-mail).

## Repository contents

```
leave-workflow/
├─ pom.xml
└─ src/main/
   ├─ java/edu/srmist/leave/
   │  ├─ LeaveWorkflowApplication.java   (@SpringBootApplication + @Deployment)
   │  ├─ api/LeaveController.java        (POST /leave starts an instance)
   │  └─ worker/LeaveWorkers.java        (two @JobWorker methods)
   └─ resources/
      ├─ application.yaml                (SaaS credentials)
      └─ bpmn/
         ├─ student-leave.bpmn
         ├─ leave-routing.dmn
         └─ fa-approval.form
```

## Names that must match

| In the model | Value              | Must match in code                                  |
|--------------|--------------------|-----------------------------------------------------|
| Process ID   | `student-leave`    | `bpmnProcessId("student-leave")` in `LeaveController` |
| Task type    | `validate-leave`   | `@JobWorker(type = "validate-leave")`               |
| Task type    | `notify-student`   | `@JobWorker(type = "notify-student")`               |
| Decision ID  | `leave-routing`    | Deployed via `@Deployment` (no Java code needed)    |
| Form ID      | `fa-approval-form` | Form file listed in `@Deployment`                   |
| Form keys    | `approved`, `remarks` | `@Variable` names in `notifyStudent`             |

## Prerequisites

| Item           | Version / note                                         |
|----------------|--------------------------------------------------------|
| Camunda account| Free trial or org account at [console.camunda.io](https://console.camunda.io) |
| JDK            | 17 or later (`java -version`)                          |
| Maven          | 3.9+ (`mvn -v`), or the IDE's bundled Maven            |
| IDE            | IntelliJ IDEA Community or VS Code with Java extensions|
| HTTP client    | Postman, or `curl`                                     |

The project uses Spring Boot 3.5.6 and `camunda-spring-boot-starter` 8.8.x. Set `camunda.version` in `pom.xml` to the same minor version as your cluster.

## Setup

### 1. Create a cluster and API client (Camunda Console)

1. Sign in at console.camunda.io.
2. **Clusters → Create new cluster**: name it `leave-demo`, pick the nearest region and generation **8.8**. Wait for **Healthy**.
3. Open the cluster → **API** tab → **Create new client**: name `leave-app`, tick the **Zeebe / Orchestration Cluster** scope.
4. Copy the **Client ID**, **Client Secret**, **Cluster ID** and **Region** (Spring Boot tab). The secret is shown only once. If you lose it, delete the client and create a new one.

> Never commit the secret to GitHub. Each group should create its own client.

### 2. Configure the app

Edit `src/main/resources/application.yaml`:

```yaml
camunda:
  client:
    mode: saas
    auth:
      client-id: <client id>
      client-secret: <client secret>
    cloud:
      cluster-id: <cluster id>
      region: <region, e.g. bru-2>
```

To keep secrets out of Git, use environment variables instead: `CAMUNDA_CLIENT_ID`, `CAMUNDA_CLIENT_SECRET`, `CAMUNDA_CLUSTER_ID`, `CAMUNDA_CLUSTER_REGION`.

### 3. Deploy the models from code

All three files are deployed by the app on start-up. No need to click Deploy in Modeler.

```java
@SpringBootApplication
@Deployment(resources = {
    "classpath:bpmn/student-leave.bpmn",
    "classpath:bpmn/leave-routing.dmn",
    "classpath:bpmn/fa-approval.form"})
public class LeaveWorkflowApplication { ... }
```

### 4. Run

```bash
mvn spring-boot:run
```

The log should show the deployment of `student-leave` and the two workers opening.

## Key code

**Job workers** (`LeaveWorkers`): the type links a method to a service task, `@Variable` reads process variables, and the returned `Map` is written back as variables.

```java
@JobWorker(type = "validate-leave")
public Map<String, Object> validateLeave(@Variable String regNo,
                                         @Variable String fromDate,
                                         @Variable String toDate) {
    LocalDate from = LocalDate.parse(fromDate);
    LocalDate to = LocalDate.parse(toDate);
    long days = ChronoUnit.DAYS.between(from, to) + 1;
    boolean valid = days > 0 && !from.isBefore(LocalDate.now());
    boolean autoApproved = valid && days <= 2;
    return Map.of("leaveDays", days, "valid", valid, "approved", autoApproved);
}

@JobWorker(type = "notify-student")
public Map<String, Object> notifyStudent(@Variable String studentName,
        @Variable String regNo, @Variable Boolean approved, @Variable String remarks) {
    String status = Boolean.TRUE.equals(approved) ? "APPROVED" : "REJECTED";
    log.info("Mail to {} ({}): your leave is {}. Remarks: {}", studentName, regNo, status, remarks);
    return Map.of("leaveStatus", status);
}
```

**REST trigger** (`LeaveController`):

```java
@PostMapping("/leave")
public Map<String, Object> applyLeave(@RequestBody LeaveRequest request) {
    ProcessInstanceEvent instance = camundaClient.newCreateInstanceCommand()
        .bpmnProcessId("student-leave")
        .latestVersion()
        .variables(request)
        .send().join();
    return Map.of("processInstanceKey", instance.getProcessInstanceKey());
}
```

## Testing

Use dates in the future, in `yyyy-MM-dd` format.

**Short leave (auto-approved)**

```bash
curl -X POST http://localhost:8080/leave \
  -H "Content-Type: application/json" \
  -d '{"studentName":"Priya","regNo":"RA2311003010001","fromDate":"2026-10-20","toDate":"2026-10-21","reason":"Medical appointment"}'
```

The instance completes immediately in Operate and the log prints `Mail to Priya ... APPROVED`.

**Long leave (needs Faculty Advisor)**: change `toDate` to `2026-10-24`. The token waits at *Faculty Advisor approval*. In Tasklist, open the task, **Assign to me**, tick **Approve this leave**, add a remark and **Complete task**.

**Invalid dates**: send a `fromDate` in the past. The DMN returns `REJECT` and the student is told REJECTED with no advisor involvement.

In **Operate**, open each instance to see the path taken and the Variables panel (`leaveDays`, `valid`, `approved`, `route`, `remarks`).

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| 401 Unauthorized at start-up | Wrong client ID/secret, or the client lacks the Zeebe / Orchestration Cluster scope. Recreate the client. |
| UNAVAILABLE / cannot connect | Wrong `cluster-id` or `region`, or addresses set by hand. In SaaS mode remove any `grpc-address` / `rest-address`. |
| Instance stuck at a service task | Worker type doesn't match the task type in the model (check spelling and case). |
| Incident: "Text cannot be parsed" | Date not in `yyyy-MM-dd` format. Fix the variable in Operate, then Retry. |
| Incident: decision `leave-routing` not found | `leave-routing.dmn` is missing from `@Deployment`. |
| Expected to find process `student-leave` | Process ID in the model differs, or the app hasn't deployed yet. |
| User task has no form | Form ID not linked, or the `.form` file is missing from `@Deployment`. |
| Old tutorial code doesn't compile | It uses `ZeebeClient` / `io.camunda.zeebe.spring.*`. Use `CamundaClient` and `io.camunda.client.annotation.*`. |

## Extension ideas

- Timer boundary event on the FA task: escalate to the HOD after 2 days.
- Send a real e-mail with the SendGrid or REST connector instead of the log line.
- Leave-balance check worker that throws a BPMN error when the balance is exhausted.
- `GET /leave/{key}` endpoint returning the instance status.
- Edit the DMN table (for example, change the 2-day limit) and redeploy to see the routing change with no Java changes.
