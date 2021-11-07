#!/usr/bin/env groovy
//
// Default Jenkinsfile - used to setup master & PR jobs from a single point
//
// (every github repo should get a Jenkins Pipeline project that follows master,
// that pipeline job triggers this pipeline, which sets up the other jobs associated
// with the project - i.e. PR validation piplines & pipelines to deploy master)
//

def GIT_URL = env.GIT_URL ? env.GIT_URL : scm.getUserRemoteConfigs()[0].getUrl()
def REPO_OWNER = GIT_URL.split('/')[3]
// trim the trailing '.git'
def REPO = GIT_URL.split('/')[4].replaceAll("\\.git\$", "")
def BASE_NAME = "${REPO_OWNER}/${REPO}"

def AXON_JENKINS_SHARED_LIBRARY = GIT_URL
def default_branch = "main"

pipeline {
  agent any

  stages {
    stage('INFO') {
      steps {
        echo "Pipeline variables"
        echo "------------------"
        echo "BASE_NAME = ${BASE_NAME}"
        echo "GIT_URL = ${GIT_URL}"
        echo "REPO_OWNER = ${REPO_OWNER}"
        echo "REPO = ${REPO}"
        echo ""

        sh label: 'Environment Variables', script: 'env | sort'
      }
    }
    stage('Folders') {
      steps {
        jobDsl scriptText: """
          folder("${REPO_OWNER}") {}:
        }
        """
        // Create the project folder, setting a shared pipeline library so that
        // simple jobs like pipelineJob can use libraries
        jobDsl scriptText: """
          folder("${BASE_NAME}") {
              properties {
                folderLibraries {
                  libraries {
                    libraryConfiguration {
                      name('axon-shared-library')
                      defaultVersion("${default_branch}")
                      implicit(false)
                      allowVersionOverride(true)
                      includeInChangesets(true)
                      retriever {
                        modernSCM {
                          scm {
                            git {
                              id('random')
                              remote("${AXON_JENKINS_SHARED_LIBRARY}")
                              credentialsId('git')
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }
        }
        """
      }
    }

    stage('Jobs') {
      steps {
        // Multibranch pipeline used for PR validation & feedback
        jobDsl scriptText: """
            multibranchPipelineJob("${BASE_NAME}/PRs") {
              properties {
                folderLibraries {
                  libraries {
                    libraryConfiguration {
                      name('axon-shared-library')
                      defaultVersion("${default_branch}")
                      implicit(false)
                      allowVersionOverride(true)
                      includeInChangesets(true)
                      retriever {
                        modernSCM {
                          scm {
                            git {
                              id('random')
                              remote("${AXON_JENKINS_SHARED_LIBRARY}")
                              credentialsId('git')
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }

              branchSources {
                branchSource {
                  source {
                    github {
                      // a unique id
                      id("${BASE_NAME}/PRs")
                      // apiUri('https://git.taservs.net/api/v3')
                      credentialsId('git')
                      repoOwner("${REPO_OWNER}")
                      repository("${REPO}")
                      repositoryUrl("${GIT_URL}")
                      configuredByUrl(false)
                      traits {
                        disableStatusUpdateTrait {}
                        gitHubPullRequestDiscovery {
                            strategyId(1)
                        }
                      }
                    }
                  }
                  configure {
                    it / sources / data / "jenkins.branch.BranchSource" <<  source (class: "org.jenkinsci.plugins.github_branch_source.GitHubSCMSource")  {
                      traits {
                        "com.adobe.jenkins.disable__github__multibranch__status.DisableStatusUpdateTrait" {}
                      }
                    }
                  }
                }
              }

              factory {
                workflowBranchProjectFactory {
                  scriptPath("Jenkinsfile.PR")
                }
              }
              triggers {
                cron('@hourly')
              }
              orphanedItemStrategy {
                discardOldItems {
                  daysToKeep(7)
                  numToKeep(20)
                }
              }
            }
        """

        // single-branch pipeline for drift/apply
        jobDsl scriptText: """
            pipelineJob("${BASE_NAME}/${default_branch}") {
              properties {
                pipelineTriggers {
                  triggers {
                    githubPush()
                    cron {
                      spec('H 0 * * *')
                    }
                  }
                }
              }
              definition {
                cpsScm {
                  lightweight(false)
                  scriptPath("Jenkinsfile.${default_branch}")
                  scm {
                    git {
                      branch("${env.GIT_BRANCH}")
                      // default to the branch that this pipeline is being run from
                      //branch("origin/${default_branch}")
                      remote {
                        credentials('git')
                        url('${GIT_URL}')
                      }
                    }
                  }
                }
              }
            }
        """

        // parameterised job that's triggered from other jobs
        // we declare the parameters here, as they are discovered on the first execution of the pipeline
        jobDsl scriptText: """
            pipelineJob("${BASE_NAME}/terraform") {
              definition {
                cpsScm {
                  lightweight(false)
                  scriptPath('Jenkinsfile.azure_terraform')
                  scm {
                    git {
                      // default to the branch that this pipeline is being run from
                      branch("${env.GIT_BRANCH}")
                      remote {
                        credentials('git')
                        url("${GIT_URL}")
                      }
                    }
                  }
                }
              }
            }
        """
        // trigger the job we've just created/updated so that parameters are loaded - this is why the defaults should result in a no-op
        // propagate: false - because we don't really care what happend, we just want to load the pipeline
        build propagate: false, job: "${BASE_NAME}/terraform"
      }
    }
  }
}
