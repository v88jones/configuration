# Jenkins Analytics

A role that sets up Jenkins for scheduling analytics tasks.

This role performs the following steps:

* Installs Jenkins using `jenkins_master`.
* Configures `config.xml` to enable security and use
  Linux Auth Domain.
* Creates Jenkins credentials.
* Enables the use of Jenkins CLI.
* Installs a seed job from configured repository, launches it and waits
  for it to finish.
* The seed job creates the analytics task jobs.

Each analytics task job is created using a task-specific DSL script which
determines the structure of the Jenkins job, e.g. its scheduled frequency, the
git repos cloned to run the task, the parameters the job requires, and the
shell script used to run the analytics task.  These DSL scripts live in a
separate git repo, configured by `ANALYTICS_SCHEDULE_JOBS_DSL_REPO_*`.


## Configuration

When you are using vagrant you **need** to set `VAGRANT_JENKINS_LOCAL_VARS_FILE`
environment variable. This variable must point to a file containing
all required variables from this section.

This file needs to contain, at least, the following variables
(see the next few sections for more information about them):

* `JENKINS_ANALYTICS_USER_PASSWORD_PLAIN`.
  See [Jenkins User Password](#jenkins-user-password) for details.
* (`JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID` and `JENKINS_ANALYTICS_GITHUB_KEY`)
  and/or `JENKINS_ANALYTICS_CREDENTIALS`.
  See [Jenkins Credentials](#jenkins-credentials) for details.
* `ANALYTICS_SCHEDULE_SECURE_REPO_*` and `ANALYTICS_SCHEDULE_<TASK_NAME>_EXTRA_VARS`.
  See [Jenkins Seed Job Configuration](#jenkins-seed-job-configuration) for details.

### End-user editable configuration

#### Jenkins user password

You'll need to override default `jenkins` user password, please do that
as this sets up the **shell** password for this user.

You'll need to set a plain password so ansible can reach Jenkins via the command line tool.

* `JENKINS_ANALYTICS_USER_PASSWORD_PLAIN`: plain password

#### Jenkins credentials

Jenkins contains its own credential store. To fill it with credentials,
please use the `JENKINS_ANALYTICS_CREDENTIALS` variable. This variable
is a list of objects, each object representing a single credential.
For now passwords and ssh-keys are supported.

If you only need credentials to access github repositories
you can override `JENKINS_ANALYTICS_GITHUB_KEY`,
which should contain contents of private key used for
authentication to checkout github repositories.

Each credential has a unique ID, which is used to match
the credential to the task(s) for which it is needed

Examples of credentials variables:

    JENKINS_ANALYTICS_GITHUB_KEY: "{{ lookup('file', 'path to keyfile')

    JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID: "github-readonly-key"

    JENKINS_ANALYTICS_CREDENTIALS:
      # id is a scope-unique credential identifier
      - id: test-password
        # Scope must be global. To have other scopes you'll need to modify addCredentials.groovy
        scope: GLOBAL
        # Username associated with this password
        username: jenkins
        type: username-password
        description: Autogenerated by ansible
        password: 'password'
      # id is a scope-unique credential identifier
      - id: "{{ JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID }}"
        scope: GLOBAL
        # Username this ssh-key is attached to
        username: git
        # Type of credential, see other entries for example
        type: ssh-private-key
        passphrase: null
        description: Autogenerated by ansible
        privatekey: "{{ JENKINS_ANALYTICS_GITHUB_KEY }}'

#### Jenkins seed job configuration

The seed job creates the Analytics Jobs that will run the analytics tasks.  By
default, the seed job creates all the available Analytics Jobs, but you can disable
these jobs, and set their parameters, using `ANALYTICS_SCHEDULE_<TASK_NAME>_*`.

Currently supported analytics tasks are:

* `ANSWER_DISTRIBUTION`: invokes
  `edx.analytics.tasks.answer_dist.AnswerDistributionWorkflow` via the
  `AnswerDistributionWorkflow.groovy` DSL.
* `IMPORT_ENROLLMENTS_INTO_MYSQL`: invokes
  `edx.analytics.tasks.enrollments.ImportEnrollmentsIntoMysql` via the
  `ImportEnrollmentsIntoMysql.groovy` DSL.
* `COURSE_ACTIVITY_WEEKLY`: invokes
  `edx.analytics.tasks.user_activity.CourseActivityWeeklyTask` via the
  `CourseActivityWeeklyTask.groovy` DSL.
* `INSERT_TO_MYSQL_ALL_VIDEO`: invokes
  `edx.analytics.tasks.video.InsertToMysqlAllVideoTask` via the
  `InsertToMysqlAllVideoTask.groovy` DSL.
* `INSERT_TO_MYSQL_COURSE_ENROLL_BY_COUNTRY:` invokes
  `edx.analytics.tasks.location_per_course.InsertToMysqlCourseEnrollByCountryWorkflow` via the
  `InsertToMysqlCourseEnrollByCountryWorkflow.groovy` DSL.

Since running the analytics tasks on EMR requires confidential ssh keys, the
convention is to store them in a secure repo, which is then cloned when running
the seed job.  To use a secure repo, override
`ANALYTICS_SCHEDULE_SECURE_REPO_URL` and
`ANALYTICS_SCHEDULE_SECURE_REPO_VERSION`.

For example:

    ANALYTICS_SCHEDULE_SECURE_REPO_URL: "git@github.com:open-craft/analytics-sandbox-private.git"
    ANALYTICS_SCHEDULE_SECURE_REPO_VERSION: "customer-analytics-schedule"

The seed job also clones a second repo, which contains the DSL scripts that
contain the analytics task DSLs.  That repo is configured using
`ANALYTICS_SCHEDULE_JOBS_DSL_REPO_*`, and it will be cloned directly into the
seed job workspace.

**Note:** There are two ways to specify a ssl-based github repo URL.  Note the
subtle difference in the paths: `github.com:your-org` vs. `github.com/your-org`.

* git@github.com:your-org/private-repo.git ✓
* ssh://git@github.com/your-org/private-repo.git ✓

*Not like this:*

* git@github.com/your-org/private-repo.git ❌
* ssh://git@github.com:your-org/private-repo.git ❌

The full list of seed job configuration variables is:

* `ANALYTICS_SCHEDULE_SECURE_REPO_URL`: Optional URL for the git repo that contains the
  analytics task schedule configuration file.  If set, Jenkins will clone this
  repo when the seed job is run.  Default is `null`.
* `ANALYTICS_SCHEDULE_SECURE_REPO_VERSION`: Optional branch/tagname to checkout
  for the secure repo.  Default is `master`.
* `ANALYTICS_SCHEDULE_SECURE_REPO_DEST`: Optional target dir for the the secure
  repo clone, relative to the seed job workspace.  Default is `analytics-secure-config`.
* `ANALYTICS_SCHEDULE_SECURE_REPO_CREDENTIAL_ID`: Credential id with read
  access to the secure repo.  Default is `{{ JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID }}`.
  See [Jenkins Credentials](#jenkins-credentials) below for details.
* `ANALYTICS_SCHEDULE_JOBS_DSL_REPO_URL`: Optional URL for the git repo that contains the analytics job DSLs.
  Default is `git@github.com:edx-ops/edx-jenkins-job-dsl.git`.
  This repo is cloned directly into the seed job workspace.
* `ANALYTICS_SCHEDULE_JOBS_DSL_REPO_VERSION`: Optional branch/tagname to checkout for the job DSL repo.
  Default is `master`.
* `ANALYTICS_SCHEDULE_JOBS_DSL_REPO_CREDENTIAL_ID`: Credential id with read access to the job DSL repo.
  Default is `{{ JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID }}`.
  See [Jenkins Credentials](#jenkins-credentials) below for details.
* `ANALYTICS_SCHEDULE_JOBS_DSL_CLASSPATH`: Optional additional classpath jars
  and dirs required to run the job DSLs.
  Each path must be newline-separated, and relative to the seed job workspace.
  Default is:

        src/main/groovy
        lib/*.jar

* `ANALYTICS_SCHEDULE_JOBS_DSL_TARGET_JOBS`: DSLs for the top-level seed job to run on build.
  Default is `jobs/analytics-edx-jenkins.edx.org/*Jobs.groovy`


* `ANALYTICS_SCHEDULE_<TASK_NAME>`:  `true`|`false`.  Must be set to `true` to create the analytics task.
* `ANALYTICS_SCHEDULE_<TASK_NAME>_FREQUENCY`: Optional string representing how
  often the analytics task should be run.  Uses a modified cron syntax, e.g.
  `@daily`, `@weekly`, see [stackoverflow](http://stackoverflow.com/a/12472740)
  for details.  Set to empty string to disable cron.
  Default is different for each analytics task.
* `ANALYTICS_SCHEDULE_<TASK_NAME>_EXTRA_VARS`: YML config or @file location to
  override the analytics task parameters.
  Consult the individual analytics task DSL for details on the options and defaults.

For example:

    ANALYTICS_SCHEDULE_ANSWER_DISTRIBUTION: true
    ANALYTICS_SCHEDULE_ANSWER_DISTRIBUTION_EXTRA_VARS: "@path/to/answer_dist_defaults.yml"

    ANALYTICS_SCHEDULE_IMPORT_ENROLLMENTS_INTO_MYSQL: true
    ANALYTICS_SCHEDULE_IMPORT_ENROLLMENTS_INTO_MYSQL_EXTRA_VARS:
      TASKS_REPO: "https://github.com/open-craft/edx-analytics-pipeline.git"
      TASKS_BRANCH: "analytics-sandbox"
      CONFIG_REPO: "https://github.com/open-craft/edx-analytics-configuration.git"
      CONFIG_BRANCH: "analytics-sandbox"
      JOB_NAME: "ImportEnrollmentsIntoMysql"
      JOB_FREQUENCY: "@monthly"
      CLUSTER_NAME: "AnswerDistribution"
      EMR_EXTRA_VARS: "@/home/jenkins/emr-vars.yml"  # see [EMR Configuration](#emr-configuration)
      FROM_DATE: "2016-01-01"
      TASK_USER: "hadoop"
      NOTIFY_EMAIL_ADDRESSES: "staff@example.com

##### EMR Configuration

The `EMR_EXTRA_VARS` parameter for each analytics task is passed by the analytics
task shell command to the ansible playbook for provisioning and terminating the
EMR cluster.

Because `EMR_EXTRA_VARS` passes via the shell, it may reference other analytics
task parameters as shell variables, e.g. `$S3_PACKAGE_BUCKET`.

**File path**

The easiest way to modify this parameter is to provide a `@/path/to/file.yml`
or `@/path/to/file.json`.  The file path must be absolute, e.g.,

    ANALYTICS_SCHEDULE_IMPORT_ENROLLMENTS_INTO_MYSQL_EXTRA_VARS:
      EMR_EXTRA_VARS: '@/home/jenkins/emr-vars.yml'

Or relative to the analytics-configuration repo cloned by the analytics task, e.g.,

    ANALYTICS_SCHEDULE_IMPORT_ENROLLMENTS_INTO_MYSQL_EXTRA_VARS:
      EMR_EXTRA_VARS: '@./config/emr-vars.yml'

To use a path relative to the analytics task workspace, build an absolute path
using the `$WORKSPACE` variable provided by Jenkins, e.g.,

    ANALYTICS_SCHEDULE_IMPORT_ENROLLMENTS_INTO_MYSQL_EXTRA_VARS:
      EMR_EXTRA_VARS: '@$WORKSPACE/../AnalyticsSeedTask/analytics-secure-config/emr-vars.yml'


**Raw JSON**

The other option, utilised by the DSL `EMR_EXTRA_VARS` default value, is to use a
JSON string.  Take care to use a *JSON string*, not raw JSON itself, as YAML is
a JSON superset, and we don't want the JSON to be parsed by ansible.

Also, because formatting valid JSON is difficult, be sure to run the text
through a JSON validator before deploying.

As with file paths, the JSON text can use analytics task parameters as shell
variables, e.g.,

    ANALYTICS_SCHEDULE_IMPORT_ENROLLMENTS_INTO_MYSQL_EXTRA_VARS:
      AUTOMATION_KEYPAIR_NAME: 'analytics-sandbox'
      VPC_SUBNET_ID: 'subnet-cd1b9c94'
      EMR_LOG_BUCKET: 's3://analytics-sandbox-emr-logs'
      CLUSTER_NAME: 'Analytics EMR Cluster'
      EMR_EXTRA_VARS: |
        {
          "name": "$CLUSTER_NAME",
          "keypair_name": "$AUTOMATION_KEYPAIR_NAME",
          "vpc_subnet_id": "$VPC_SUBNET_ID",
          "log_uri": "$EMR_LOG_BUCKET"
        }

#### Other useful variables

* `JENKINS_ANALYTICS_CONCURRENT_JOBS_COUNT`: Configures number of
  executors (or concurrent jobs this Jenkins instance can
  execute). Defaults to `2`.

### General configuration

Following variables are used by this role:

Variables used by command waiting on Jenkins start-up after running
`jenkins_master` role:

    jenkins_connection_retries: 60
    jenkins_connection_delay: 0.5

#### Auth realm

Jenkins auth realm encapsulates user management in Jenkins, that is:

* What users can log in
* What credentials they use to log in

Realm type stored in `jenkins_auth_realm.name` variable.

In future we will try to enable other auth domains, while
preserving the ability to run cli.

##### Unix Realm

For now only `unix` realm supported -- which requires every Jenkins
user to have a shell account on the server.

Unix realm requires the following settings:

* `service`: Jenkins uses PAM configuration for this service. `su` is
a safe choice as it doesn't require a user to have the ability to login
remotely.
* `plain_password`:  plaintext password, **you must change** default values.

Example realm configuration:

    jenkins_auth_realm:
      name: unix
      service: su
      plain_password: jenkins


#### Seed job configuration

Seed job is configured in `jenkins_seed_job` variable, which has the following
attributes:

* `name`:  Name of the job in Jenkins.
* `time_trigger`: A Jenkins cron entry defining how often this job should run.
* `removed_job_action`: what to do when a job created by a previous run of seed job
  is missing from current run. This can be either `DELETE` or`IGNORE`.
* `removed_view_action`: what to do when a view created by a previous run of seed job
  is missing from current run. This can be either `DELETE` or`IGNORE`.
* `scm`: Scm object is used to define seed job repository and related settings.
  It has the following properties:
  * `scm.type`: It must have value of `git`.
  * `scm.url`: URL for the repository.
  * `scm.credential_id`: Id of a credential to use when authenticating to the
    repository.
    This setting is optional. If it is missing or falsy, credentials will be omitted.
    Please note that when you use ssh repository url, you'll need to set up a key regardless
    of whether the repository is public or private (to establish an ssh connection
    you need a valid public key).
  * `scm.target_jobs`: A shell glob expression relative to repo root selecting
    jobs to import.
  * `scm.additional_classpath`: A path relative to repo root, pointing to a
     directory that contains additional groovy scripts used by the seed jobs.

Example scm configuration:

    jenkins_seed_job:
      name: seed
      time_trigger: "H * * * *"
      removed_job_action: "DELETE"
      removed_view_action: "IGNORE"
      scm:
        type: git
        url: "git@github.com:edx-ops/edx-jenkins-job-dsl.git"
        credential_id: "github-deploy-key"
        target_jobs: "jobs/analytics-edx-jenkins.edx.org/*Jobs.groovy"
        additional_classpath: "src/main/groovy"

Known issues
------------

1. Playbook named `execute_ansible_cli.yaml`, should be converted to an
   Ansible module (it is already used in a module-ish way).
2. Anonymous user has discover and get job permission, as without it
   `get-job`, `build <<job>>` commands wouldn't work.
   Giving anonymous these permissions is a workaround for
   transient Jenkins issue (reported [couple][1] [of][2] [times][3]).
3. We force unix authentication method -- that is, every user that can login
   to Jenkins also needs to have a shell account on master.


Dependencies
------------

- `jenkins_master`

[1]: https://issues.jenkins-ci.org/browse/JENKINS-12543
[2]: https://issues.jenkins-ci.org/browse/JENKINS-11024
[3]: https://issues.jenkins-ci.org/browse/JENKINS-22143