defaults = [
  action_type: ['print_branches'],
  version: '',
  protect_branch: false
]

isMaster  = BRANCH_NAME == 'master'
isDevelop = BRANCH_NAME == 'develop'
isHotfix  = BRANCH_NAME.startsWith('hotfix')
isRelease = BRANCH_NAME.startsWith('release')

if (isMaster || isDevelop) {
  defaults.action_type.add('start_release')
  defaults.version = '0.0.0'
  defaults.protect_branch = true
  if (isMaster) {
    defaults.release_type = 'hotfix'
  } else if (isDevelop) {
    defaults.release_type = 'release'
  }
} else if (isHotfix || isRelease) {
  defaults.action_type.addAll(['merge_release', 'finish_release', 'unprotect_release'])
}

reposList = [
  [owner: 'heatray', name: 'foo', dir: 'foo'],
  [owner: 'heatray', name: 'bar', dir: 'bar']
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
      description:  "Action type\nmaster > hotfix\ndevelop > release",
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
    stage('Prepare') {
      steps {
        script {
          if (params.wipe) {
            deleteDir()
            checkout scm
          }
          // utils = load 'utils.groovy'
          curBranch = BRANCH_NAME
          newBranch = defaults.release_type + '/v' + params.vesion
          baseBranches = []
          // reposList = utils.getReposList()
          stats = [
            repos: [:],
            list: '',
            success: 0,
            total: reposList.size()
          ]
          reposList.each {
            stats.repos.put(it.owner + '/' + it.name, [:])
          }
        }
      }
    }
    stage('Stages') {
      parallel {
        stage('Print Branches') {
          when {
            expression { params.action_type == 'print_branches' }
            beforeAgent true
          }
          steps {
            script {
              Boolean printed
              stats.repos.each { repo, status ->
                printed = printBranches(repo)
                status.primary = (printed) ? 'success' : 'failure'
              }
            }
          }
        }
        stage('Start Release') {
          when {
            expression { params.action_type == 'start_release' }
            beforeAgent true
          }
          steps {
            script {
              baseBranches = [BRANCH_NAME]
              Boolean created
              Boolean locked

              stats.repos.each { repo, status ->
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  if (!checkRemoteBranch(repo, 'develop')
                    && !createBranch(repo, 'develop', 'master'))
                    error("Can't create develop branch.")

                  if (!checkRemoteBranch(repo, newBranch)) {
                    created = createBranch(repo, newBranch, baseBranches[0])
                    status.primary = (created) ? 'success' : 'failure'
                  } else {
                    echo "Branch already ${newBranch} exists."
                    status.primary = 'skip'
                  }

                  if (params.protect_branch) {
                    locked = protectBranch(repo, newBranch)
                    status.secondary = (locked) ? 'lock' : ''
                  }
                }
              }
            }
          }
        }
        stage('Merge Release') {
          when {
            expression { params.action_type == 'merge_release' }
            beforeAgent true
          }
          steps {
            script {
              baseBranches = ['master']
              Boolean merged

              stats.repos.each { repo, status ->
                dir ('repos/' + repo) {
                  if (checkRemoteBranch(repo, curBranch)) {
                    checkoutRepo(repo)
                    merged = mergeBranch(repo, curBranch, baseBranches)
                    status.primary = (merged) ? 'success' : 'failure'
                  } else {
                    echo "Branch doesn't ${curBranch} exist."
                    status.primary = 'skip'
                  }
                }
              }
            }
          }
        }
        stage('Finish Release') {
          when {
            expression { params.action_type == 'finish_release' }
            beforeAgent true
          }
          steps {
            script {
              baseBranches = ['master', 'develop']
              if (!params.extra_branch.isEmpty())
                baseBranches.add(params.extra_branch)
              Boolean unlocked
              Boolean merged
              Boolean deleted

              stats.repos.each { repo, status ->
                dir ('repos/' + repo) {
                  if (checkRemoteBranch(repo, curBranch)) {
                    checkoutRepo(repo)
                    merged = mergeBranch(repo, curBranch, baseBranches)
                    status.primary = (merged) ? 'success' : 'failure'
                    println merged
                    if (merged) {
                      unprotectBranch(repo, curBranch)
                      deleted = deleteBranch(repo, curBranch)
                      status.secondary = (deleted) ? 'delete' : ''
                    }
                  } else {
                    echo "Branch doesn't ${curBranch} exist."
                    status.primary = 'skip'
                  }
                }
              }
            }
          }
        }
        stage('Unprotect Release') {
          when {
            expression { params.action_type == 'unprotect_release' }
            beforeAgent true
          }
          steps {
            script {
              Boolean unlock
              stats.repos.each { repo, status ->
                unlock = unprotectBranch(repo, curBranch)
                status.primary = (unlock) ? 'success' : 'failure'
              }
            }
          }
        }
      }
    }
  }
  post {
    success {
      script {
        println "test"

        String icons
        stats.repos.each { repo, status ->
          icons = ''
          switch(status.primary) {
            case 'skip':    icons += '🆗'; stats.success++; break
            case 'success': icons += '✅'; stats.success++; break
            case 'failure': icons += '❎'; break
          }
          switch(status.secondary) {
            case 'lock':    icons += '🔒'; break
            case 'delete':  icons += '🗑';  break
          }
          stats.list += '\n' + icons + ' ' + repo
        }

        println stats.list
        println currentBuild.result

        if (stats.success == 0)
          currentBuild.result = 'FAILURE'
        else if (stats.success != stats.total)
          currentBuild.result = 'UNSTABLE'
        else if (stats.success == stats.total)
          currentBuild.result = 'SUCCESS'

        println currentBuild.result

        if ((params.action_type == 'start_release'
          || params.action_type == 'merge_release'
          || params.action_type == 'finish_release')
          && params.notify && stats.success > 0)
          sendNotification()
        else if (params.action_type == 'print_branches'
          || params.action_type == 'unprotect_release')
          println stats.success + '/' + stats.total + '\n' + stats.list.trim()
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
    """,
    returnStatus: true
  ) == 0
}

def mergeBranch(String repo, String branch, ArrayList baseBranches) {
  Integer ret = sh (
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
      [[ \$merged -ne \${#base_branches[@]} ]] && exit 2
    """,
    returnStatus: true
  )
  println "ret: $ret"
  return ret == 0
}

def deleteBranch(String repo, String branch) {
  return sh (
    label: "${repo}: delete ${branch}",
    script: """
      git branch -D ${branch}
      git push --delete origin ${branch}
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

def printBranches(String repo) {
  return sh (
    label: "${repo}: branches list",
    script: """
      gh api -X GET repos/${repo}/branches?per_page=100 | \
      jq -c '.[] | { name, protected }'
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
  String text
  switch(params.action_type) {
    case 'start_release':
      text = "Branch `${curBranch}` created from `${baseBranches[0]}`"
      break
    case 'merge_release':
      text = "Branch `${curBranch}` merged into `${baseBranches[0]}`"
      break
    case 'finish_release':
      text = "Branch `${curBranch}` merged into "
      text += baseBranches.collect({"`$it`"}).join(', ')
      break
  }
  text += " \\[${stats.success}/${stats.total}]${stats.list}"

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
}
