# HyperExecute Assignment — Notes

Repo: `vinr0753/testng-selenium-hyperexecute-sample`
Fixed YAML: [`yaml/win/v1/(TestNG)Fixme.yaml`](./yaml/win/v1/(TestNG)Fixme.yaml)
Workflow: [`.github/workflows/main.yml`](./.github/workflows/main.yml)

---

## Task 1: Fix the broken YAML

The original YAML failed for more than one reason — issues surfaced in stages as each was
fixed and the job was re-run.

### Issues found and why they broke the job

- **`env: TOKEN: anvdegtod-asdaasda0asda-asda`** — invalid YAML. A nested key can't sit on
  the same line as its parent. This threw a hard parse error
  (`mapping values are not allowed here`) before the job could even start.
  **Fix:** moved `TOKEN` onto its own indented line under `env:`.

- **`retryOnFailure: true` had a stray leading space** — every other top-level key sits at
  column 0; this one had one space of indent. **Fix:** removed the leading space.

- **`conCurrency: 1`** (typo'd casing) — HyperExecute's schema expects `concurrency`
  (lowercase). Because the key didn't match, it was silently ignored, and since
  `autosplit: true` requires `concurrency` to be defined, the job failed validation with
  `Invalid yaml content: Concurrency is required in autosplit mode`.
  **Fix:** corrected to `concurrency: 1`.

- **`(TestNG)Fixme.yaml` config path had literal parentheses**, and the workflow step
  referenced it *unquoted* in the `--config` flag:
  ```bash
  --config yaml/win/v1/(TestNG)Fixme.yaml
  ```
  In bash, `(` opens a subshell — unquoted, this is a shell metacharacter, not just a
  filename character, and threw `syntax error near unexpected token '('`.
  **Fix:** quoted the path: `--config "yaml/win/v1/(TestNG)Fixme.yaml"`.

- **JDK mismatch** — `pom.xml` sets `maven.compiler.target: 11`, but the machine actually
  compiling had JDK 8 installed (`openjdk version "1.8.0_442"`), giving
  `Fatal error compiling: invalid target release: 11`.
  **Fix:** added a `runtime:` block to the YAML so HyperExecute provisions JDK 11 on its
  own VM for the test-execution stage:
  ```yaml
  runtime:
    language: java
    version: 11
  ```

### Evidence
- Job link: `https://hyperexecute.lambdatest.com/hyperexecute/task?jobId=<job-id>`
- ![Job running successfully on the dashboard](./images/task1-dashboard.png)

---

## Task 2: Environment variables

### YAML
```yaml
env:
  TOKEN: anvdegtod-asdaasda0asda-asda
  ENVIRONMENT: staging
pre:
  - mvn -Dmaven.repo.local=./.m2 dependency:resolve
  - echo TOKEN=%TOKEN%
  - echo ENVIRONMENT=%ENVIRONMENT%
```

**Note on shell syntax:** `pre` on this Windows runner executes under `cmd.exe`, not bash —
confirmed by testing. `$TOKEN` (bash-style) silently resolved to nothing; `%TOKEN%`
(`cmd.exe`-style) worked correctly and printed the real value.

### Reading the variable in test code
`src/test/java/Test1.java`:
```java
public static String token = System.getenv("TOKEN");
public static String environment = System.getenv("ENVIRONMENT");
```
Printed inside `@BeforeMethod`:
```java
System.out.println("TOKEN=" + token);
System.out.println("ENVIRONMENT=" + environment);
```

### Evidence
- `pre` step log output:
  ```
  TOKEN=anvdegtod-asdaasda0asda-asda
  ENVIRONMENT=staging
  ```
- Test execution console output:
  ```
  TOKEN=anvdegtod-asdaasda0asda-asda
  ENVIRONMENT=staging
  ```
- ![Env values printed in pre step and test output](./images/task2-env-output.png)

---

## Task 3: Force a failure and configure retries

### The intentional failure
`src/test/java/Test1.java`, inside `test1_element_addition_1`, using the test's own
already-computed values so the failure reads as a genuine check rather than an arbitrary
`1 == 2`:

```java
// Intentional hard failure for retryOnFailure testing.
// Uses the test's own real computed values but forces a mismatch,
// so the failure is deterministic on every run.
Assert.assertEquals(actualText, expectedText + " (forced mismatch)",
        "Intentional failure for retry testing: expected [" + expectedText
                + " (forced mismatch)] but got [" + actualText + "]");
```

### YAML
```yaml
retryOnFailure: true
maxRetries: 1
```

### Evidence
Job log shows the test failing, then retrying, then failing again on the retry (this
particular assertion is designed to always fail, so the *retry firing* is the thing being
verified — not the test ultimately passing):

```
x [1]  "Test_1" (1m17s)
x [1]  {retry 1} "Test_1" (58s)
✔ [1]  "Test_2" (42s)
✔ [1]  "Test_3" (51s)
✔ [1]  "Test_4" (35s)
```

`{retry 1}` confirms `retryOnFailure` triggered exactly as configured. The overall job
status shows `FAILED` because this test is designed to always fail — that's expected and
intentional for this task, not a regression elsewhere in the suite.

- ![Retry evidence from job log](./images/task3-retry.png)

---

## Task 4: Linux/Unix basics

Two inputs used deliberately:
- **`hyperexecute-cli.log`** — the real HyperExecute CLI log from the Task 1 job run, for `grep`
- **`test-data/deploy_status.txt`** — a purpose-built, clean space-delimited sample file,
  for `awk`/`sed`/the pipe chain (see note below on why)

### 1. grep — find FAIL/ERROR lines in the real job log
```bash
grep -iE 'FAIL|ERROR' hyperexecute-cli.log
```
*What it does:* `-E` enables extended regex so `|` means "or"; `-i` makes it case-insensitive.
Prints every line containing `FAIL` or `ERROR`.

Sample output (from the actual job log):
```
{"level":"info","time":"...","msg":"\n FAILED \n"}
{"level":"info","time":"...","msg":"    Failed test stage percentage:         25.00%"}
{"level":"error","time":"...","msg":"Exiting with error: Failed tasks found."}
```

### 2. awk — extract column 2
`test-data/deploy_status.txt`:
```
app-gateway staging PASS
auth-service staging FAIL
payment-service production PASS
user-service staging PASS
notification-service production FAIL
search-service staging PASS
inventory-service production PASS
reporting-service staging FAIL
```

```bash
awk '{print $2}' test-data/deploy_status.txt
```
*What it does:* splits each line on whitespace and prints the second field — here, the
deployment environment for each service.

Output:
```
staging
staging
production
staging
production
staging
production
staging
```

### 3. sed — find and replace
```bash
sed 's/staging/production/g' test-data/deploy_status.txt
```
*What it does:* `s/old/new/g` substitutes every occurrence of `staging` with `production`
on each line (`g` = all matches, not just the first).

Output:
```
app-gateway production PASS
auth-service production FAIL
payment-service production PASS
user-service production PASS
notification-service production FAIL
search-service production PASS
inventory-service production PASS
reporting-service production FAIL
```

### 4. Chained pipe — grep then awk
```bash
grep 'FAIL' test-data/deploy_status.txt | awk '{print $1}'
```
*What it does:* filters down to only the failed deployments, then extracts just the
service name (column 1) from those lines — a genuinely useful "which services failed"
report.

Output:
```
auth-service
notification-service
reporting-service
```

---

## Repo layout
```
.
├── .github/workflows/main.yml
├── yaml/win/v1/(TestNG)Fixme.yaml
├── src/test/java/Test1.java
├── test-data/deploy_status.txt
└── images/
    ├── task1-dashboard.png
    ├── task2-env-output.png
    └── task3-retry.png
```
