// Do not trigger build regularly on change requests as it costs a lot
String cronTrigger = ''
if(env.BRANCH_NAME == "master") {
  cronTrigger = '10 0 * * 5'
}

env.MAVEN_NTP = true

properties([
  // disableConcurrentBuilds(abortPrevious: true),
  // buildDiscarder(logRotator(numToKeepStr: '7')),
  buildDiscarder(logRotator(numToKeepStr: '20')),
  pipelineTriggers([cron(cronTrigger)])
])

if (env.BRANCH_NAME == 'master' && currentBuild.buildCauses*._class == ['jenkins.branch.BranchEventCause']) {
  currentBuild.result = 'NOT_BUILT'
  error 'No longer running builds on response to master branch pushes. If you wish to cut a release, use “Re-run checks” from this failing check in https://github.com/jenkinsci/bom/commits/master'
}

def mavenEnv(Map params = [:], Closure body) {
  def attempt = 0
  def attempts = 6
  retry(count: attempts, conditions: [kubernetesAgent(handleNonKubernetes: true), nonresumable()]) {
    echo 'Attempt ' + ++attempt + ' of ' + attempts
    // no Dockerized tests; https://github.com/jenkins-infra/documentation/blob/master/ci.adoc#container-agents
    node('maven-bom') {
      timeout(120) {
        withChecks(name: 'Tests', includeStage: true) {
          infra.withArtifactCachingProxy {
            withEnv([
              'JAVA_HOME=/opt/jdk-' + params['jdk'],
              'PATH+JDK=/opt/jdk-' + params['jdk'] + '/bin',
              "MAVEN_ARGS=${env.MAVEN_ARGS != null ? MAVEN_ARGS : ''} -B ${env.MAVEN_NTP != null ? '-ntp' : ''} -Dmaven.repo.local=${WORKSPACE_TMP}/m2repo",
              "MVN_LOCAL_REPO=${WORKSPACE_TMP}/m2repo",
            ]) {
              infra.loadMavenLocalCacheIfAny(env.MVN_LOCAL_REPO)

              body()
            }
          }
          if (junit(testResults: '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml').failCount > 0) {
            // TODO JENKINS-27092 throw up UNSTABLE status in this case
            error 'Some test failures, not going to continue'
          }
        }
      }
    }
  }
}

@NonCPS
def parsePlugins(plugins) {
  def pluginsByRepository = [:]
  plugins.each { plugin ->
    def splits = plugin.split('\t')
    pluginsByRepository[splits[0].split('/')[1]] = splits[1].split(',')
  }
  pluginsByRepository
}

def pluginsByRepository
def lines
def fullTestMarkerFile
def weeklyTestMarkerFile
def durations = [:]

// debug
<<<<<<< HEAD
// set to a job name to skip prep.sh and retrieve content from last success
def retrieveArchiveFrom = ''
||||||| parent of 0f1ab58a (debug: retrieve prep from prep-only job for now)
def retrieveArchiveFrom = ''
=======
def retrieveArchiveFrom = 'Tools/bom/prep-only'
>>>>>>> 0f1ab58a (debug: retrieve prep from prep-only job for now)
def runPrep = true
// set to [] to use plugins.txt
def limitedPluginSet = [
  'jenkinsci/aws-credentials-plugin	aws-credentials',
  'jenkinsci/aws-global-configuration-plugin	aws-global-configuration',
  'jenkinsci/azure-credentials-plugin	azure-credentials',
  'jenkinsci/azure-keyvault-plugin	azure-keyvault',
  'jenkinsci/azure-sdk-plugin	azure-sdk',
  'jenkinsci/azure-storage-plugin	windows-azure-storage',
  'jenkinsci/badge-plugin	badge',
  'jenkinsci/basic-branch-build-strategies-plugin	basic-branch-build-strategies',
  'jenkinsci/cron_column-plugin	cron_column',
  'jenkinsci/pipeline-maven-plugin	pipeline-maven,pipeline-maven-api,pipeline-maven-database',
]

stage('prep') {
  mavenEnv(jdk: 21) {
    if (retrieveArchiveFrom) {
      try {
        copyArtifacts(
          projectName: retrieveArchiveFrom,
          selector: lastSuccessful(),
          filter: 'target.tar.gz',
        )
        // TODO: warn if not same commit
        sh 'tar -xzf target.tar.gz && rm target.tar.gz && ls . && ls target'
        runPrep = false
      } catch (e) {
        echo 'WARNING: no archive found from ' + retrieveArchiveFrom
      }
    }
    if (runPrep) {
      checkout scm
      withEnv(['SAMPLE_PLUGIN_OPTS=-Dset.changelist']) {
        sh '''
        mvn -v
        bash prep.sh
        '''
      }
    }
    // debug
    archiveArtifacts 'target/*.txt'
    fullTestMarkerFile = fileExists 'full-test'
    weeklyTestMarkerFile = fileExists 'weekly-test'
    def plugins = readFile('target/plugins.txt').split('\n')
    if (limitedPluginSet) {
      echo 'INFO: using limitedPluginSet'
      plugins = limitedPluginSet
    }
    pluginsByRepository = parsePlugins(plugins)

    lines = readFile('target/lines.txt').split('\n')
    lines = [lines[0], lines[-1]] // Save resources by running PCT only on newest and oldest lines
    lines.each { line ->
      stash name: line, includes: "pct.sh,excludes.txt,bom-*/excludes.txt,target/pct.jar,target/megawar-${line}.war"
    }
    // debug
    echo "${pluginsByRepository.size()} repositories:\n${plugins.join('\n')}"
    echo "${lines.size()} lines: ${lines.join(' ')} "
    infra.prepareToPublishIncrementals()
  }
}

if (BRANCH_NAME == 'master' || fullTestMarkerFile || weeklyTestMarkerFile || env.CHANGE_ID && (pullRequest.labels.contains('full-test') || pullRequest.labels.contains('weekly-test'))) {
  def branches = [failFast: false]
  lines.each {line ->
    if (line != 'weekly' && (weeklyTestMarkerFile || env.CHANGE_ID && pullRequest.labels.contains('weekly-test'))) {
      return
    }
    pluginsByRepository.each { repository, plugins ->
      branches["pct-$repository-$line"] = {
        def jdk = line == 'weekly' || line == '2.555.x' ? 21 : 17
        mavenEnv(jdk: jdk) {
          unstash line
          withEnv([
            "PLUGINS=${plugins.join(',')}",
            "LINE=$line",
            'EXTRA_MAVEN_PROPERTIES=maven.test.failure.ignore=true:surefire.rerunFailingTestsCount=1'
          ]) {
            def start = System.currentTimeMillis()
            try {
              sh '''
              mvn -v
              bash pct.sh
              '''
            } catch (e) {
              if (!(e instanceof InterruptedException) && !(e instanceof org.jenkinsci.plugins.workflow.support.steps.AgentOfflineException)) {
                unstable('PCT failed in ' + repository + ' - line ' + line)
              } else {
                throw e
              }
            } finally {
              def elapsed = System.currentTimeMillis() - start
              durations["pct-$repository-$line"] = (elapsed / 1000.0)
            }
          }
        }
      }
    }
  }
  parallel branches
  stage('duration report') {
    node('maven-bom') {
      Double totalTime = 0
      def reportLines = ''
      durations.each { branch, time ->
        totalTime += time as Double
        reportLines += '<testcase name="' + branch + '" classname="pct-duration.' + branch + '" time="' + time + '"/>\n'
      }
      if (reportLines) {
        def content = """<?xml version="1.0" encoding="UTF-8"?>
          <testsuite name="bom" time="${totalTime}">
          ${reportLines}
          </testsuite>
        """
        writeFile file: 'bom-report.xml', text: content
        archiveArtifacts artifacts: 'bom-report.xml'
        junit allowEmptyResults: true, testResults: 'bom-report.xml'
      }
    }
  }
}

if (fullTestMarkerFile) {
  error 'Remember to `git rm full-test` before taking out of draft'
}

infra.maybePublishIncrementals()
