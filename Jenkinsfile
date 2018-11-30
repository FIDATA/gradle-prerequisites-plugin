#!/usr/bin/env groovy
/*
 * Jenkinsfile for gradle-prerequisites-plugin
 * Copyright © 2018  Basil Peace
 *
 * This file is part of gradle-prerequisites-plugin.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
 * implied. See the License for the specific language governing
 * permissions and limitations under the License.
 */
import org.jfrog.hudson.pipeline.types.ArtifactoryServer
import org.jfrog.hudson.pipeline.types.GradleBuild
import org.jfrog.hudson.pipeline.types.buildInfo.BuildInfo

@SuppressWarnings(['UnusedVariable', 'NoDef', 'VariableTypeRequired'])
@Library('jenkins-pipeline-shared-library@v1.1.0') dummy

properties([
  disableConcurrentBuilds()
])

node {
  GradleBuild rtGradle

  stage ('Checkout') {
    List<Map<String, ? extends Serializable>> extensions = [
      [$class: 'WipeWorkspace'],
      [$class: 'CloneOption', noTags: false, shallow: false],
    ]
    if (!env.CHANGE_ID) {
      extensions.add([$class: 'LocalBranch', localBranch: env.BRANCH_NAME])
    }
    checkout([
      $class: 'GitSCM',
      branches: scm.branches,
      doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
      extensions: extensions,
      userRemoteConfigs: scm.userRemoteConfigs,
    ])
    gitAuthor()
  }

  ArtifactoryServer server = Artifactory.server 'FIDATA'
  rtGradle = Artifactory.newGradleBuild()
  rtGradle.useWrapper = true
  rtGradle.usesPlugin = true

  /*
   * WORKAROUND:
   * Disabling Gradle Welcome message
   * should be done in fidata_build_toolset.
   * See https://github.com/FIDATA/infrastructure/issues/85
   * <grv87 2018-09-21>
   */
  /*
   * WORKAROUND:
   * Gradle can't provide console with colors but no other rich features.
   * So, we use plain console for now
   * https://github.com/gradle/gradle/issues/6843
   * <grv87 2018-09-21>
   */
  /*
   * WORKAROUND:
   * Build cache should be turned on in gradle.properties
   * as soon as we move sensitive properties to separate place
   * and put gradle.properties under version control
   * <grv87 2018-09-22>
   */
  String gradleSwitches = '-Dorg.gradle.internal.launcher.welcomeMessageEnabled=false --console=plain --info --warning-mode all --full-stacktrace --build-cache'

  ansiColor {
    withGpgScope("${ pwd() }/.scoped-gpg", 'GPG', 'GPG_KEY_PASSWORD') { String fingerprint ->
      withEnv([
        "ORG_GRADLE_PROJECT_gpgKeyId=$fingerprint",
      ]) {
        withCredentials([
          usernamePassword(credentialsId: 'Github 2', usernameVariable: 'ORG_GRADLE_PROJECT_gitUsername', passwordVariable: 'ORG_GRADLE_PROJECT_gitPassword'),
          string(credentialsId: 'Github', variable: 'ORG_GRADLE_PROJECT_ghToken'),
          usernamePassword(credentialsId: 'Artifactory', usernameVariable: 'ORG_GRADLE_PROJECT_artifactoryUser', passwordVariable: 'ORG_GRADLE_PROJECT_artifactoryPassword'),
          string(credentialsId: 'GPG_KEY_PASSWORD', variable: 'ORG_GRADLE_PROJECT_gpgKeyPassphrase'),
          usernamePassword(credentialsId: 'Bintray', usernameVariable: 'ORG_GRADLE_PROJECT_bintrayUser', passwordVariable: 'ORG_GRADLE_PROJECT_bintrayAPIKey'),
        ]) {
          BuildInfo buildInfo
          try {
            stage('Clean') {
              timeout(time: 5, unit: 'MINUTES') {
                buildInfo = rtGradle.run tasks: 'clean', switches: gradleSwitches
                /*
                 * TODO:
                 * Move these filters into separate library
                 * <grv87 2018-09-22>
                 */
                buildInfo.env.filter.clear()
                buildInfo.env.filter.addExclude('*Password')
                buildInfo.env.filter.addExclude('*Passphrase')
                buildInfo.env.filter.addExclude('*SecretKey')
                buildInfo.env.filter.addExclude('*SECRET_KEY')
                buildInfo.env.filter.addExclude('*APIKey')
                buildInfo.env.filter.addExclude('*_API_KEY')
                buildInfo.env.filter.addExclude('*gradlePluginsKey')
                buildInfo.env.filter.addExclude('*gradlePluginsSecret')
                buildInfo.env.filter.addExclude('*OAuthClientSecret')
                buildInfo.env.filter.addExclude('*Token')
                buildInfo.env.collect()
              }
            }
            stage('Assemble') {
              try {
                timeout(time: 5, unit: 'MINUTES') {
                  rtGradle.run tasks: 'assemble', switches: gradleSwitches, buildInfo: buildInfo
                }
              } finally {
                warnings(
                  consoleParsers: [
                    [parserName: 'Java Compiler (javac)'],
                    [parserName: 'JavaDoc Tool'],
                  ]
                )
              }
            }
            stage('Check') {
              try {
                timeout(time: 15, unit: 'MINUTES') {
                  rtGradle.run tasks: 'check', switches: gradleSwitches, buildInfo: buildInfo
                }
              } finally {
                warnings(
                  consoleParsers: [
                    [parserName: 'Java Compiler (javac)'],
                    [parserName: 'JavaDoc Tool'],
                  ]
                )
                junit(
                  testResults: 'build/reports/**/*.xml',
                  allowEmptyResults: true,
                  keepLongStdio: true,
                )
                // TODO
                junit(
                  testResults: 'build/test-results/**/*.xml',
                  allowEmptyResults: true,
                  keepLongStdio: true,
                )
                publishHTML(target: [
                  reportName: 'CodeNarc',
                  reportDir: 'build/reports/html/codenarc',
                  reportFiles: [
                    'main',
                    'test',
                    'functionalTest',
                    'compatTest',
                    'buildSrc',
                  ].collect { "${ it }.html" }.join(', '), // TODO: read from directory ?
                  allowMissing: true,
                  keepAll: true,
                  alwaysLinkToLastBuild: false /* TODO: determine if we are on develop HEAD */
                ])
                publishHTML(target: [
                  reportName: 'Test',
                  reportDir: 'build/reports/html/test',
                  reportFiles: 'index.html',
                  allowMissing: true,
                  keepAll: true,
                  alwaysLinkToLastBuild: false /* TODO: determine if we are on develop HEAD */
                ])
                publishHTML(target: [
                  reportName: 'FunctionalTest',
                  reportDir: 'build/reports/html/functionalTest',
                  reportFiles: 'index.html',
                  allowMissing: true,
                  keepAll: true,
                  alwaysLinkToLastBuild: false /* TODO: determine if we are on develop HEAD */
                ])
                publishHTML(target: [
                  reportName: 'CompatTest',
                  reportDir: 'build/reports/html/compatTest',
                  reportFiles:
                    readFile(file: '.stutter/java8.lock', encoding: 'UTF-8') // TODO: respect other Java versions
                      .split('[\r\n]+')
                      // Copy of algorithm from StutterExtension.getLockedVersions
                      .findAll { !it.startsWith('#') }
                      .collect { "${ it.trim() }/index.html" }
                      .join(', '),
                  allowMissing: true,
                  keepAll: true,
                  alwaysLinkToLastBuild: false /* TODO: determine if we are on develop HEAD */
                ])
              }
            }
            stage('Release') {
              try {
                timeout(time: 5, unit: 'MINUTES') {
                  rtGradle.run tasks: 'release', switches: gradleSwitches, buildInfo: buildInfo
                }
              } finally {
                warnings(
                  consoleParsers: [
                    [parserName: 'Java Compiler (javac)'],
                    [parserName: 'JavaDoc Tool'],
                  ]
                )
              }
            }
          } finally {
            server.publishBuildInfo buildInfo
          }
        }
      }
    }
  }
}
