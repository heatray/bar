defaults = [
  action_type: ['print_branches'],
  action_help: '',
  version: '',
  protect_branch: false
]

if (BRANCH_NAME == 'master' || BRANCH_NAME == 'develop') {
  defaults.action_type.add('start_release')
  if (BRANCH_NAME == 'master') {
    defaults.release_type = 'hotfix'
  } else if (BRANCH_NAME == 'develop') {
    defaults.release_type = 'release'
  }
  defaults.action_help = " (start for ${defaults.release_type} on current branch)"
  defaults.version = '0.0.0'
  defaults.protect_branch = true
} else if (BRANCH_NAME.startsWith('hotfix') || BRANCH_NAME.startsWith('release')) {
  defaults.action_type.addAll(['merge_release', 'finish_release', 'unprotect_release'])
}

reposList = [
  [owner: 'heatray', name: 'foo', dir: 'foo']
]

pipeline {
  agent { label 'linux_64' }
  parameters {
    booleanParam (
      name:         'wipe',
      description:  'Wipe out current workspace',
      defaultValue: false
    )
    choice (
      name:         'action_type',
      description:  "Action type" + defaults.action_help,
      choices:      defaults.action_type
    )
    string (
      name:         'vesion',
      description:  'Release version (for start_release only)',
      defaultValue: defaults.version
    )
    booleanParam (
      name:         'protect_branch',
      description:  'Protect branch (for start_release only)',
      defaultValue: defaults.protect_branch
    )
    string (
      name:         'extra_branch',
      description:  'Extra branch (for finish_release only)',
      defaultValue: ''
    )
    booleanParam (
      name:         'notify',
      description:  'Telegram notification',
      defaultValue: true
    )
  }
  stages {
    stage('Branch Manager') {
      steps {
        script {
          echo "BRANCH_NAME=" + BRANCH_NAME
          echo "CHANGE_ID=" + CHANGE_ID
          echo "CHANGE_URL=" + CHANGE_URL
          echo "CHANGE_TITLE=" + CHANGE_TITLE
          echo "CHANGE_AUTHOR=" + CHANGE_AUTHOR
          echo "CHANGE_AUTHOR_DISPLAY_NAME=" + CHANGE_AUTHOR_DISPLAY_NAME
          echo "CHANGE_AUTHOR_EMAIL=" + CHANGE_AUTHOR_EMAIL
          echo "CHANGE_TARGET=" + CHANGE_TARGET
          echo "CHANGE_BRANCH=" + CHANGE_BRANCH
          echo "CHANGE_FORK=" + CHANGE_FORK
          echo "TAG_NAME=" + TAG_NAME
          echo "TAG_TIMESTAMP=" + TAG_TIMESTAMP
          echo "TAG_UNIXTIME=" + TAG_UNIXTIME
          echo "TAG_DATE=" + TAG_DATE
          echo "BUILD_NUMBER=" + BUILD_NUMBER
          echo "BUILD_ID=" + BUILD_ID
          echo "BUILD_DISPLAY_NAME=" + BUILD_DISPLAY_NAME
          echo "JOB_NAME=" + JOB_NAME
          echo "JOB_BASE_NAME=" + JOB_BASE_NAME
          echo "BUILD_TAG=" + BUILD_TAG
          echo "EXECUTOR_NUMBER=" + EXECUTOR_NUMBER
          echo "NODE_NAME=" + NODE_NAME
          echo "NODE_LABELS=" + NODE_LABELS
          echo "WORKSPACE=" + WORKSPACE
          echo "WORKSPACE_TMP=" + WORKSPACE_TMP
          echo "JENKINS_HOME=" + JENKINS_HOME
          echo "JENKINS_URL=" + JENKINS_URL
          echo "BUILD_URL=" + BUILD_URL
          echo "JOB_URL=" + JOB_URL
          echo "GIT_COMMIT=" + GIT_COMMIT
          echo "GIT_PREVIOUS_COMMIT=" + GIT_PREVIOUS_COMMIT
          echo "GIT_PREVIOUS_SUCCESSFUL_COMMIT=" + GIT_PREVIOUS_SUCCESSFUL_COMMIT
          echo "GIT_BRANCH=" + GIT_BRANCH
          echo "GIT_LOCAL_BRANCH=" + GIT_LOCAL_BRANCH
          echo "GIT_CHECKOUT_DIR=" + GIT_CHECKOUT_DIR
          echo "GIT_URL=" + GIT_URL
          echo "GIT_COMMITTER_NAME=" + GIT_COMMITTER_NAME
          echo "GIT_AUTHOR_NAME=" + GIT_AUTHOR_NAME
          echo "GIT_COMMITTER_EMAIL=" + GIT_COMMITTER_EMAIL
          echo "GIT_AUTHOR_EMAIL=" + GIT_AUTHOR_EMAIL

          if (params.wipe) {
            deleteDir()
            checkout scm
          }
          currentBuild.displayName += ' - ' + params.action_type

          stats = [repos: [:]]
          String branch = BRANCH_NAME
          ArrayList baseBranches = []
          // def utils = load 'utils.groovy'
          // utils.getReposList().each {
          reposList.each {
            stats.repos.put(it.owner + '/' + it.name, [:])
          }
          Boolean pAction
          Boolean sAction

          if (params.action_type == 'print_branches') {

            stats.repos.each { repo, status ->
              pAction = printBranches(repo)
              status.primary = (pAction) ? 'success' : 'failure'
            }

          } else if (params.action_type == 'start_release') {

            branch = defaults.release_type + '/v' + params.vesion
            baseBranches = [BRANCH_NAME]

            stats.repos.each { repo, status ->
              if (checkRemoteBranch(repo, branch)) {
                echo "${repo}: Branch already ${branch} exists."
                status.primary = 'skip'
              } else {
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  if (!checkRemoteBranch(repo, 'develop')
                    && !createBranch(repo, 'develop', 'master'))
                    error("Can't create develop branch.")

                  pAction = createBranch(repo, branch, baseBranches[0])
                  status.primary = (pAction) ? 'success' : 'failure'
                }
              }

              if (params.protect_branch) {
                sAction = protectBranch(repo, branch)
                status.secondary = (sAction) ? 'lock' : ''
              }
            }

          } else if (params.action_type == 'merge_release') {

            baseBranches = ['master']

            stats.repos.each { repo, status ->
              if (!checkRemoteBranch(repo, branch)) {
                echo "${repo}: Branch doesn't ${branch} exist."
                status.primary = 'skip'
              } else {
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  pAction = mergeBranch(repo, branch, baseBranches)
                  status.primary = (pAction) ? 'success' : 'failure'
                }
              }
            }

          } else if (params.action_type == 'finish_release') {

            baseBranches = ['master', 'develop']
            if (!params.extra_branch.isEmpty())
              baseBranches.add(params.extra_branch)

            stats.repos.each { repo, status ->
              if (!checkRemoteBranch(repo, branch)) {
                echo "${repo}: Branch doesn't ${branch} exist."
                status.primary = 'skip'
              } else {
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  pAction = mergeBranch(repo, branch, baseBranches)
                  status.primary = (pAction) ? 'success' : 'failure'
                  if (pAction) {
                    unprotectBranch(repo, branch)
                    if (repo != 'ONLYOFFICE/documents-pipeline') {
                      sAction = deleteBranch(repo, branch)
                      status.secondary = (sAction) ? 'delete' : ''
                    }
                  }
                }
              }
            }

          } else if (params.action_type == 'unprotect_release') {

            stats.repos.each { repo, status ->
              pAction = unprotectBranch(repo, branch)
              status.primary = (pAction) ? 'success' : 'failure'
            }

          }

          stats.putAll([
            branch: branch,
            baseBranches: baseBranches,
            success: stats.repos.findAll { repo, status ->
              status.primary == 'skip' || status.primary == 'success'
            }.size(),
            total: stats.repos.size()
          ])
          println stats

        }
      }
    }
  }
  post {
    success {
      script {
        if (stats.success == 0)
          currentBuild.result = 'FAILURE'
        else if (stats.success != stats.total)
          currentBuild.result = 'UNSTABLE'
        else if (stats.success == stats.total)
          currentBuild.result = 'SUCCESS'

        sendNotification()
      }
    }
  }
}

def checkoutRepo(String repo, String branch = 'master') {
  sh (
    label: "${repo}: checkout",
    script: """
      if [ "\$(GIT_DIR=.git git rev-parse --is-inside-work-tree)" = 'true' ]; then
        git fetch --all --prune
        git checkout -f ${branch}
        git reset --hard origin/${branch}
        git clean -df
      else
        rm -rfv ./*
        git clone -b ${branch} git@github.com:${repo}.git .
      fi
      git branch -vv
    """
  )
}

def checkRemoteBranch(String repo, String branch = 'master') {
  return sh (
    label: "${repo}: check branch ${branch}",
    script: "git ls-remote --exit-code git@github.com:${repo}.git ${branch}",
    returnStatus: true
  ) == 0
}

def createBranch(String repo, String branch, String baseBranch) {
  return sh (
    label: "${repo}: start ${branch}",
    script: """
      git checkout ${baseBranch}
      git reset --hard origin/${baseBranch}
      git checkout -B ${branch}
      git push origin ${branch}
      git branch -vv
      echo "Branch created."
    """,
    returnStatus: true
  ) == 0
}

def mergeBranch(String repo, String branch, ArrayList baseBranches) {
  return sh (
    label: "${repo}: merge ${branch} into ${baseBranches.join(' ')}",
    script: """#!/bin/bash -xe
      git checkout ${branch}
      git reset --hard origin/${branch}
      base_branches=(${baseBranches.join(' ')})
      merged=0
      rev_branch=\$(git rev-parse @)
      for base in "\${base_branches[@]}"; do
        git checkout \$base
        git reset --hard origin/\$base
        rev_base=\$(git rev-parse @)
        if [[ \$rev_branch == \$rev_base ]]; then
          ((++merged))
          echo "No new commits."
          continue
        fi
        gh pr create --repo ${repo} --base \$base --head ${branch} \
          --title "Merge branch ${branch} into \$base" --fill || \
        true
        git merge ${branch} --no-edit --no-ff \
          -m "Merge branch ${branch} into \$base" || \
        continue
        git push origin \$base
        ((++merged))
      done
      git branch -vv
      if [[ \$merged -ne \${#base_branches[@]} ]]; then
        echo "Not fully merged."
        exit 2
      fi
      echo "Branch merged."
    """,
    returnStatus: true
  ) == 0
}

def deleteBranch(String repo, String branch) {
  return sh (
    label: "${repo}: delete ${branch}",
    script: """
      git branch -D ${branch}
      git push --delete origin ${branch}
      echo "Branch deleted."
    """,
    returnStatus: true
  ) == 0
}

def printBranches(String repo) {
  return sh (
    label: "${repo}: branches list",
    script: """
      gh api -X GET repos/${repo}/branches?per_page=100 | \
      jq -r '.[] | [.name, .protected] | @tsv'
    """,
    returnStatus: true
  ) == 0
}

def protectBranch(String repo, String branch) {
  return sh (
    label: "${repo}: protect ${branch}",
    script: """
      echo '{
        "required_status_checks": null,
        "enforce_admins": true,
        "required_pull_request_reviews": null,
        "restrictions": {
          "users": [],
          "teams": []
        }
      }' | \
      gh api -X PUT repos/${repo}/branches/${branch}/protection --input -
    """,
    returnStatus: true
  ) == 0
}

def unprotectBranch(String repo, String branch) {
  return sh (
    label: "${repo}: unprotect ${branch}",
    script: "gh api -X DELETE repos/${repo}/branches/${branch}/protection",
    returnStatus: true
  ) == 0
}

def sendNotification() {
  Integer chatId = -579429080 // test
  String text = ''
  switch(params.action_type) {
    case 'start_release':
      text = "Branch `${stats.branch}` created from `${stats.baseBranches[0]}`"
      break
    case 'merge_release':
      text = "Branch `${stats.branch}` merged into `${stats.baseBranches[0]}`"
      break
    case 'finish_release':
      text = "Branch `${stats.branch}` merged into "
      text += stats.baseBranches.collect({"`$it`"}).join(', ')
      break
    default:
      text = 'Stats'
      break
  }
  text += " \\[${stats.success}/${stats.total}]"
  stats.repos.each { repo, status ->
    text += '\n'
    switch(status.primary) {
      case 'skip':    text += '🆗'; break
      case 'success': text += '✅'; break
      case 'failure': text += '❎'; break
    }
    switch(status.secondary) {
      case 'lock':    text += '🔒'; break
      case 'delete':  text += '♻️'; break
    }
    text += ' ' + repo
  }

  if (params.notify && stats.success > 0
    && (params.action_type == 'start_release'
    || params.action_type == 'merge_release'
    || params.action_type == 'finish_release')) {
    withCredentials([string(
      credentialsId: 'telegram-bot-token',
      variable: 'TELEGRAM_TOKEN'
    )]) {
      sh label: "Send Telegram Notification", script: "curl -X POST -s -S \
        -d parse_mode=markdown \
        -d chat_id=${chatId} \
        --data-urlencode text='${text}' \
        https://api.telegram.org/bot\$TELEGRAM_TOKEN/sendMessage"
    }
  } else {
    echo text
  }
}
