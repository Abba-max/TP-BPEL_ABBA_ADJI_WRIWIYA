# TP-BPEL - Student Enrollment (Orchestration & Choreography)

This repository contains the implementation of the Student Enrollment workflow for TP4:

* **Part A (Orchestration):** Four contract-first JAX-WS SOAP services (`university`, `bank`, `residence`, `transport`) and the `EnrollmentProcess` WS-BPEL 2.0 process deployed on Apache ODE 1.3.8.
* **Part B (Choreography):** Four Spring Boot JMS event adapters exchanging JSON events over Apache ActiveMQ Artemis 2.33.x multicast topics.
* **Included deliverables:**
  * `bpel-orchestrator/`: `EnrollmentProcess.bpel`, `EnrollmentProcess.wsdl`, `types/enrollment.xsd`, and `deploy.xml`
  * `enrollment-tests.xml`: SoapUI 5.7.x project with 4 test cases
  * `report/`: PDF and Word report plus execution screenshots

---

## Prerequisites & Auto-Detected Environment

* **JDK 11** (`java -version` -> `11.0.x`)
* **Apache Maven 3.9.x** (`mvn -version`)
* **Apache Tomcat 9.0.x** with `ode.war` (Apache ODE 1.3.8) extracted into `webapps/ode/`
* **Apache ActiveMQ Artemis 2.33.x**
* **SoapUI 5.7.x**

Run the setup block for your OS once per terminal inside `tp-bpel-enrollment/`. It automatically detects `JAVA_HOME`, `mvn`, `TOMCAT_HOME` / `CATALINA_HOME`, and your Artemis broker (checking local `..\..\tools\` / `../../tools/` first, then standard `~/opt/tomcat9`, `~/artemis-broker`, and system `PATH`):

* **macOS:**

  ```bash
  [ -d "../../tools/jdk-11/Contents/Home/bin" ] && export JAVA_HOME="$(cd ../../tools/jdk-11/Contents/Home && pwd)" && export PATH="$JAVA_HOME/bin:$PATH"
  [ -d "../../tools/maven-3.9/bin" ] && export PATH="$(cd ../../tools/maven-3.9/bin && pwd):$PATH"
  export TOMCAT_HOME="${TOMCAT_HOME:-${CATALINA_HOME:-$([ -d ../../tools/tomcat-9 ] && cd ../../tools/tomcat-9 && pwd || echo "$HOME/opt/tomcat9")}}"
  export ARTEMIS_BIN="$( ([ -x ../../tools/artemis-broker/bin/artemis ] && cd ../../tools/artemis-broker/bin && echo "$(pwd)/artemis") || ([ -x "$HOME/artemis-broker/bin/artemis" ] && echo "$HOME/artemis-broker/bin/artemis") || command -v artemis )"
  ```

* **Windows (PowerShell):**

  ```powershell
  if (Test-Path "..\..\tools\jdk-11\bin\java.exe") { $env:JAVA_HOME = (Resolve-Path "..\..\tools\jdk-11").Path; $env:Path = "$env:JAVA_HOME\bin;$env:Path" }
  if (Test-Path "..\..\tools\maven-3.9\bin\mvn.cmd") { $env:Path = "$((Resolve-Path '..\..\tools\maven-3.9\bin').Path);$env:Path" }
  if (-not $env:CATALINA_HOME) { $env:CATALINA_HOME = @($env:TOMCAT_HOME, "..\..\tools\tomcat-9", "$HOME\opt\tomcat9") | Where-Object { $_ -and (Test-Path $_) } | Select-Object -First 1 | ForEach-Object { (Resolve-Path $_).Path } }
  $env:ARTEMIS_BIN = @("..\..\tools\artemis-broker\bin\artemis.cmd", "$HOME\artemis-broker\bin\artemis.cmd", (Get-Command artemis.cmd -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Source)) | Where-Object { $_ -and (Test-Path $_) } | Select-Object -First 1 | ForEach-Object { (Resolve-Path $_).Path }
  ```

---

## Part A - Build, Deploy, and Test (Orchestration)

All commands below assume your terminal is inside `tp-bpel-enrollment/`.

### 1. Build the Maven project

Same command on both **macOS** and **Windows**:

```bash
mvn clean install
```

### 2. Deploy the 4 partner WARs and the BPEL process to Tomcat / ODE

* **macOS:**

  ```bash
  cp services/university-service/target/university-service.war "$TOMCAT_HOME/webapps/"
  cp services/bank-service/target/bank-service.war "$TOMCAT_HOME/webapps/"
  cp services/residence-service/target/residence-service.war "$TOMCAT_HOME/webapps/"
  cp services/transport-service/target/transport-service.war "$TOMCAT_HOME/webapps/"

  rm -rf "$TOMCAT_HOME/webapps/ode/WEB-INF/processes/enrollment" "$TOMCAT_HOME/webapps/ode/WEB-INF/processes/enrollment.deployed"
  mkdir -p "$TOMCAT_HOME/webapps/ode/WEB-INF/processes/enrollment"
  cp -r bpel-orchestrator/* "$TOMCAT_HOME/webapps/ode/WEB-INF/processes/enrollment/"
  cp services/*/src/main/resources/*.wsdl "$TOMCAT_HOME/webapps/ode/WEB-INF/processes/enrollment/"
  ```

* **Windows (PowerShell):**

  ```powershell
  Copy-Item services\university-service\target\university-service.war, services\bank-service\target\bank-service.war, services\residence-service\target\residence-service.war, services\transport-service\target\transport-service.war "$env:CATALINA_HOME\webapps\" -Force

  $odeProc = "$env:CATALINA_HOME\webapps\ode\WEB-INF\processes\enrollment"
  Remove-Item "$odeProc*" -Recurse -Force -ErrorAction SilentlyContinue
  New-Item -ItemType Directory -Path $odeProc -Force | Out-Null
  Copy-Item bpel-orchestrator\* $odeProc -Recurse -Force
  Copy-Item services\*\src\main\resources\*.wsdl $odeProc -Force
  ```

### 3. Start Tomcat 9

* **macOS:**

  ```bash
  "$TOMCAT_HOME/bin/catalina.sh" run
  ```

* **Windows (PowerShell):**

  ```powershell
  & "$env:CATALINA_HOME\bin\catalina.bat" run
  ```

### 4. Verify and run the SoapUI test suite

1. Confirm the partner WSDLs and ODE process are live:
   * `http://localhost:8080/ode/`
   * `http://localhost:8080/ode/processes/EnrollmentService?wsdl`
2. Open **SoapUI 5.7.x**, click **File -> Import Project**, and select `enrollment-tests.xml`.
3. Open **`EnrollmentTestSuite`** and click **Run** to execute all 4 test cases:
   * `TC1_Nominal_Enrollment` (`350000` -> `ENROLLED`)
   * `TC2_Refused_Payment_Compensation` (`900000` -> `REFUSED` + student cancelled at University)
   * `TC3_Boundary_500000_Approved` (`500000` -> `ENROLLED`)
   * `TC4_Boundary_500001_Refused` (`500001` -> `REFUSED`)

---

## Part B - Launch and Test (Choreography)

### 1. Create (first time only, if no broker exists) and start ActiveMQ Artemis

* **macOS:**

  ```bash
  [ -z "$ARTEMIS_BIN" ] && artemis create ~/artemis-broker --user admin --password admin --allow-anonymous && export ARTEMIS_BIN="$HOME/artemis-broker/bin/artemis"
  "$ARTEMIS_BIN" run
  ```

* **Windows (PowerShell):**

  ```powershell
  if (-not $env:ARTEMIS_BIN) { artemis.cmd create "$HOME\artemis-broker" --user admin --password admin --allow-anonymous; $env:ARTEMIS_BIN = "$HOME\artemis-broker\bin\artemis.cmd" }
  & "$env:ARTEMIS_BIN" run
  ```

### 2. Create the 4 multicast topics

Open a new terminal inside `tp-bpel-enrollment/` (run the setup block from Prerequisites first) while Artemis is running:

* **macOS:**

  ```bash
  "$ARTEMIS_BIN" address create --name EnrollmentRequested --multicast --user admin --password admin
  "$ARTEMIS_BIN" address create --name StudentRegistered --multicast --user admin --password admin
  "$ARTEMIS_BIN" address create --name PaymentDecided --multicast --user admin --password admin
  "$ARTEMIS_BIN" address create --name EnrollmentCompleted --multicast --user admin --password admin
  ```

* **Windows (PowerShell):**

  ```powershell
  & "$env:ARTEMIS_BIN" address create --name EnrollmentRequested --multicast --user admin --password admin
  & "$env:ARTEMIS_BIN" address create --name StudentRegistered --multicast --user admin --password admin
  & "$env:ARTEMIS_BIN" address create --name PaymentDecided --multicast --user admin --password admin
  & "$env:ARTEMIS_BIN" address create --name EnrollmentCompleted --multicast --user admin --password admin
  ```

### 3. Start the 4 Spring Boot choreography adapters

Open 4 separate terminals inside `tp-bpel-enrollment/` (same commands on **macOS** and **Windows**):

```bash
mvn -pl choreography/university-event-adapter spring-boot:run
mvn -pl choreography/bank-event-adapter spring-boot:run
mvn -pl choreography/residence-event-adapter spring-boot:run
mvn -pl choreography/transport-event-adapter spring-boot:run
```

Wait until all 4 windows show `Started ...EventAdapter in ... seconds`.

### 4. Trigger the Nominal and Refused test flows

* **macOS:**

  ```bash
  # Nominal flow (350,000 FCFA -> ENROLLED)
  "$ARTEMIS_BIN" producer --destination topic://EnrollmentRequested --user admin --password admin --message-count 1 --message '{"correlationId":"enr-001","type":"EnrollmentRequested","data":{"student":{"name":"Alice","program":"ISI4"},"amount":"350000"}}'

  # Refused flow + compensation (900,000 FCFA -> REFUSED)
  "$ARTEMIS_BIN" producer --destination topic://EnrollmentRequested --user admin --password admin --message-count 1 --message '{"correlationId":"enr-002","type":"EnrollmentRequested","data":{"student":{"name":"Bob","program":"ISI4"},"amount":"900000"}}'
  ```

* **Windows (PowerShell):**

  ```powershell
  # Nominal flow (350,000 FCFA -> ENROLLED)
  & "$env:ARTEMIS_BIN" --% producer --destination topic://EnrollmentRequested --user admin --password admin --message-count 1 --message "{\"correlationId\":\"enr-001\",\"type\":\"EnrollmentRequested\",\"data\":{\"student\":{\"name\":\"Alice\",\"program\":\"ISI4\"},\"amount\":\"350000\"}}"

  # Refused flow + compensation (900,000 FCFA -> REFUSED)
  & "$env:ARTEMIS_BIN" --% producer --destination topic://EnrollmentRequested --user admin --password admin --message-count 1 --message "{\"correlationId\":\"enr-002\",\"type\":\"EnrollmentRequested\",\"data\":{\"student\":{\"name\":\"Bob\",\"program\":\"ISI4\"},\"amount\":\"900000\"}}"
  ```

### 5. Verify the results

* Open the Artemis Console at `http://localhost:8161/console` (`admin` / `admin`) -> **Addresses** to see the message counts on `EnrollmentRequested`, `StudentRegistered`, `PaymentDecided`, and `EnrollmentCompleted`.
* Check the adapter terminals for `[University] Registered ...` (nominal) and `[University] Cancelled registration ...` (refused compensation).
