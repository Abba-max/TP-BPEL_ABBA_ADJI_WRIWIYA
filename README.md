# TP-BPEL - Student Enrollment (Orchestration & Choreography)

This repository contains the complete implementation of the Student Enrollment workflow:
- **Part A (Orchestration):** Four contract-first JAX-WS SOAP services (`university`, `bank`, `residence`, `transport`) and the `EnrollmentProcess` WS-BPEL 2.0 orchestrator on Apache ODE 1.3.8.
- **Part B (Choreography):** Four Spring Boot JMS event adapters communicating over Apache ActiveMQ Artemis 2.33.x multicast topics.
- **Deliverables included:**
  - `bpel-orchestrator/`: `EnrollmentProcess.bpel`, `EnrollmentProcess.wsdl`, `types/enrollment.xsd`, and `deploy.xml`
  - `enrollment-tests.xml`: SoapUI 5.7.x test suite with 4 automated test cases
  - `report/`: Final PDF report (`TP-BPEL-Report.pdf`), editable Word report (`TP-BPEL-Report.docx`), and execution screenshots

All scripts automatically detect standard installations (`$TOMCAT_HOME` / `$CATALINA_HOME` / `~/opt/tomcat9`, `~/artemis-broker`, and system `PATH`) as well as the isolated `../../tools/` folder if present.

---

## Part A - Build, Deploy, and Test (Orchestration)

### 1. Start Tomcat 9 (with `ode.war` deployed in `webapps/`)
- **macOS / Linux:**
  ```bash
  export TOMCAT_HOME=${TOMCAT_HOME:-${CATALINA_HOME:-~/opt/tomcat9}}
  $TOMCAT_HOME/bin/catalina.sh run
  ```
- **Windows (PowerShell):**
  ```powershell
  & "$env:CATALINA_HOME\bin\catalina.bat" run
  ```

### 2. Build and Deploy All 4 Partner WARs + BPEL Process
Open a second terminal in `tp-bpel-enrollment/`:
- **macOS / Linux:**
  ```bash
  chmod +x deploy-part-a.sh
  ./deploy-part-a.sh
  ```
- **Windows (PowerShell):**
  ```powershell
  .\deploy-part-a.ps1
  ```

### 3. Verify and Run the SoapUI Test Suite
1. Check that the BPEL WSDL is live: `http://localhost:8080/ode/processes/EnrollmentService?wsdl`
2. In **SoapUI 5.7.x**, click **File -> Import Project**, select `enrollment-tests.xml`, open **`EnrollmentTestSuite`**, and click **Run**:
   - `TC1_Nominal_Enrollment` (`350000` -> `ENROLLED`)
   - `TC2_Refused_Payment_Compensation` (`900000` -> `REFUSED` + student cancelled at University)
   - `TC3_Boundary_500000_Approved` (`500000` -> `ENROLLED`)
   - `TC4_Boundary_500001_Refused` (`500001` -> `REFUSED`)

---

## Part B - Launch and Test (Choreography)

### 1. Start the ActiveMQ Artemis 2.33.x Broker
- **macOS / Linux:**
  ```bash
  artemis create ~/artemis-broker --user admin --password admin --allow-anonymous
  ~/artemis-broker/bin/artemis run
  ```
- **Windows (PowerShell):**
  ```powershell
  .\artemis-broker\bin\artemis.cmd run
  ```

### 2. Create Multicast Topics and Start All 4 Adapters
From `tp-bpel-enrollment/`:
- **macOS / Linux:**
  ```bash
  chmod +x start-part-b.sh trigger-part-b.sh
  ./start-part-b.sh
  ```
- **Windows (PowerShell):**
  ```powershell
  .\start-part-b.ps1
  ```
*(The script automatically creates the 4 multicast topics `EnrollmentRequested`, `StudentRegistered`, `PaymentDecided`, and `EnrollmentCompleted` and starts all 4 Spring Boot adapters.)*

### 3. Trigger Nominal and Refused Flows
Open `http://localhost:8161/console` (`admin` / `admin`) to observe the topics, then trigger both scenarios using either `curl` or the helper script:

- **Option A - Using `curl` (macOS / Linux / Git Bash):**
  ```bash
  # Nominal flow (350,000 FCFA -> ENROLLED):
  curl -u admin:admin -H "Content-Type: application/json" \
    -XPOST http://localhost:8161/api/rest/queue/EnrollmentRequested \
    -d '{"correlationId":"enr-001","type":"EnrollmentRequested","data":{"student":{"name":"Alice","program":"ISI4"},"amount":"350000"}}'

  # Refused flow with compensation (900,000 FCFA -> REFUSED):
  curl -u admin:admin -H "Content-Type: application/json" \
    -XPOST http://localhost:8161/api/rest/queue/EnrollmentRequested \
    -d '{"correlationId":"enr-002","type":"EnrollmentRequested","data":{"student":{"name":"Bob","program":"ISI4"},"amount":"900000"}}'
  ```

- **Option B - Using the auto-detecting helper script (works on all Artemis 2.33 setups without the REST WAR):**
  - **macOS / Linux:**
    ```bash
    ./trigger-part-b.sh nominal
    ./trigger-part-b.sh refused
    ```
  - **Windows (PowerShell):**
    ```powershell
    .\trigger-part-b.ps1 -Scenario nominal
    .\trigger-part-b.ps1 -Scenario refused
    ```
