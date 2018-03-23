import groovy.json.JsonSlurperClassic

String GIT_LOG;
String GIT_LOCAL_BRANCH;

def caughtError;

node(label: 'Mobile Builder 2') {
  currentBuild.result = "SUCCESS"

  ansiColor('xterm') {
    try {

      String PATH = "PATH=$PATH:/usr/local/bin"; // this is to find the npm command
      PATH = "PATH=$PATH:/usr/local/opt/libpq/bin" // this is to find the pg_dump command

      String LOCK_SCOPE = env.NODE_NAME.replace(/ /, '-')
      lock(resource: "mobile-web-performance-lock-$LOCK_SCOPE") {

        stage('checkout workable') {
          checkout scm
        }

        stage('get commit hash') {
          COMMIT_HASH = sh (
            script: "git ls-remote --heads git@github.com:Workable/workable.git | grep \$BRANCH\$ | awk '{print \$1}'",
            returnStdout: true).trim()
        }

        stage('cleanup') {
          sh (script: "rm -rf node_modules && rm -rf tmp")
        }

        stage('database config') {
          sh (script: "cp config/database.sample.yml config/database.yml")
        }

        stage('npm install') {
          sh (script: "npm install")
        }

        stage('gem install') {
          sh (script: "gem install pg -v0.18.2 -- --with-pg-config=/Applications/Postgres.app/Contents/Versions/9.6/bin/pg_config")
          sh (script: "gem install bundler")
          sh (script: "bundle")
        }

        stage('compile assets') {
          sh (script: "npm install --save-dev webpack")
          sh (script: "node_modules/.bin/webpack")
        }

        stage('seed database') {
          // Restart postgres to avoid "PG::ObjectInUse: ERROR:  database "workable_development" is being accessed by other users"
          sh (script: "kill -9 $(ps aux | grep Postgres/var-9.6 | grep -v 'grep ' | awk '{print $2}')")
          sh (script: "postgres -D /Users/angelos/Library/Application\ Support/Postgres/var-9.6 -p 5432 &")
          sh (script: "sleep 2")
          sh (script: "QUICK=true rake db:reseed && BYPASS_MOBILE_GENERATOR_DATA=yes rake qa:generate_mobile_data")
        }
      }

    } catch (err) {

      caughtError = err;
      currentBuild.result = "FAILURE"

    } finally {
      switch(currentBuild.result) {
        case "FAILURE":
          slackit([
            channel: YODA_SLACK_CHANNEL,
            color: "danger",
            message: """
                    |${JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)\n
                    |```
                    |${BRANCH} (${COMMIT_HASH})
                    |```
                    |```${caughtError}```
                    """.stripMargin()
          ])
          if (caughtError) {
           throw caughtError; // rethrow so that the build fails
          }
          break;
        default:
          // do nothing
      }
    }
  }
}

def slackit(params) {
  if (!env.SKIP_SLACK) {
    slackSend(params)
  }
}

def sanitizeJobName(jobName) {
  return jobName.replaceAll('%2F', '/');
}