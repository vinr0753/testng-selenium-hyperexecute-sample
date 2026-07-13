# HyperExecute Assignment — Notes

Repo: `vinr0753/testng-selenium-hyperexecute-sample`
Fixed YAML: [`yaml/win/v1/(TestNG)Fixme.yaml`](./yaml/win/v1/(TestNG)Fixme.yaml)
Workflow: [`.github/workflows/main.yml`](./.github/workflows/main.yml)

---

## Task 1: Fix the broken YAML

The original YAML failed for more than one reason — issues surfaced in stages as each was
fixed and the job was re-run.

### Issues found and why they broke the job

**1. `(TestNG)Fixme.yaml` config path had literal parentheses, referenced unquoted**

The workflow step referenced the config path unquoted in the `--config` flag:
```bash
--config yaml/win/v1/(TestNG)Fixme.yaml
```
In bash, `(` opens a subshell — unquoted, this is a shell metacharacter, not just a
filename character, and threw `syntax error near unexpected token '('`.

**Fix:** quoted the path:
```bash
--config "yaml/win/v1/(TestNG)Fixme.yaml"
```

*Evidence:* [Job run #29252461233](https://github.com/vinr0753/testng-selenium-hyperexecute-sample/actions/runs/29252461233/job/86824136153)

---

**2. Workflow cloned the wrong repo**

`main.yml`'s `sampleRepoLink` input defaulted to
`https://github.com/LambdaTest/testng-selenium-hyperexecute-sample` (the original,
unmodified upstream repo), not this fork. Since `Starting CLI testing` does
`git clone ${{ github.event.inputs.sampleRepoLink }}`, any run using that default would
clone LambdaTest's pristine sample instead of this repo — meaning none of the YAML fixes,
`Test1.java` changes, or `test-data/deploy_status.txt` would actually be present, and the
job would silently be testing someone else's unmodified code instead of this submission.

**Fix:** changed the default to
`https://github.com/vinr0753/testng-selenium-hyperexecute-sample` (this fork).

*Evidence:* [Job run #29253004319](https://github.com/vinr0753/testng-selenium-hyperexecute-sample/actions/runs/29253004319/job/86825867678)

---

**3. `retryOnFailure: true` had a stray leading space**

Every other top-level key sits at column 0; this one had one space of indent, which broke
the YAML mapping structure.

**Fix:** removed the leading space.

---

**4. `conCurrency: 1` (typo'd casing)**

HyperExecute's schema expects `concurrency` (lowercase). Because the key didn't match, it
was silently ignored, and since `autosplit: true` requires `concurrency` to be defined, the
job failed validation with:
```
Invalid yaml content: Concurrency is required in autosplit mode
```

**Fix:** corrected to `concurrency: 1`.

*Evidence:*

![Comparison with sample YAML](./images/Screenshot%202026-07-13%20183935.png)

---

**5. JDK mismatch**

`pom.xml` sets `maven.compiler.target: 11`, but the machine actually compiling had JDK 8
installed (`openjdk version "1.8.0_442"`), giving:
```
Fatal error compiling: invalid target release: 11
```

**Fix:** added a `runtime:` block to the YAML so HyperExecute provisions JDK 11 on its own
VM for the test-execution stage:
```yaml
runtime:
  language: java
  version: 11
```

*Evidence:*
- [HyperExecute job log (pre-run stage)](https://hyperexecute.lambdatest.com/hyperexecute/task?activeGraph=&isArtifactOpen=&jobId=412660f7-6d78-44b4-b1ef-f25222820899&link=pre.log&logType=prerun&order=1&previousLogType=&scenario_search_text=&stageId=827865ee-3125-4e28-aede-2b3c2a0529ed&tab=&taskId=HYPW-3223818-1783950821453358601XBH&taskStatus=&viewBy=)

![Check for Java version](./images/Screenshot%202026-07-13%20225152.png)

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

*Evidence:*
- `pre` step log output:
  ```
  TOKEN=anvdegtod-asdaasda0asda-asda
  ENVIRONMENT=staging
  ```
- [HyperExecute job log (setup-runtime stage)](https://hyperexecute.lambdatest.com/hyperexecute/task?activeGraph=&isArtifactOpen=&jobId=607e6f32-0766-4535-8e1e-89b9db712ca4&link=setup-runtime.log&logType=setup-runtime&order=1&previousLogType=&scenario_search_text=&stageId=8ea646e5-9ef4-4520-910d-71744efc361b&tab=&taskId=HYPW-3223818-1783952224044930826VBM&taskStatus=&viewBy=)

![Debug output](./images/Screenshot%202026-07-13%20225311.png)

---

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

*Evidence:*
- Test execution console output:
  ```
  TOKEN=anvdegtod-asdaasda0asda-asda
  ENVIRONMENT=staging
  ```
- [HyperExecute job log (setup-runtime stage)](https://hyperexecute.lambdatest.com/hyperexecute/task?activeGraph=&isArtifactOpen=&jobId=607e6f32-0766-4535-8e1e-89b9db712ca4&link=setup-runtime.log&logType=setup-runtime&order=1&previousLogType=&scenario_search_text=&stageId=8ea646e5-9ef4-4520-910d-71744efc361b&tab=&taskId=HYPW-3223818-1783952224044930826VBM&taskStatus=&viewBy=)

![Env values printed in test output](./images/Screenshot%202026-07-13%20230849.png)

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

- [HyperExecute job log (retry scenario)](https://hyperexecute.lambdatest.com/hyperexecute/task?activeGraph=&isArtifactOpen=&jobId=0143d175-03a3-46c0-befe-61725638eec5&link=command.1.1.log&logType=scenario&order=4&previousLogType=&scenario_search_text=&stageId=4a6ec3c7-443c-49fc-a673-dd9178fb6017&tab=&taskId=HYPW-3223818-1783952908135368257MBD&taskStatus=&viewBy=)

![Retry evidence from job log](./images/Screenshot%202026-07-13%20231135.png)

---

## Task 4: Linux/Unix basics

Two inputs used deliberately:
- **`hyperexecute-cli.log`** — the real HyperExecute CLI log from the Task 1 job run, for `grep`
- **`test-data/deploy_status.txt`** — a purpose-built, clean space-delimited sample file,
  for `awk`/`sed`/the pipe chain

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

### Evidence

[Workflow run #29265796739](https://github.com/vinr0753/testng-selenium-hyperexecute-sample/actions/runs/29265796739/job/86870521243)

---

## Repo layout
```
.
├── .github/workflows/main.yml
├── yaml/win/v1/(TestNG)Fixme.yaml
├── src/test/java/Test1.java
├── test-data/deploy_status.txt
└── images/
    ├── Screenshot 2026-07-13 183935.png
    ├── Screenshot 2026-07-13 225152.png
    ├── Screenshot 2026-07-13 225311.png
    ├── Screenshot 2026-07-13 230849.png
    └── Screenshot 2026-07-13 231135.png
```
