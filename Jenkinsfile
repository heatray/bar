defaults = [
  action_type: ['print_branches'],
  action_help: '',
  protect_branch: false
]

isMaster  = BRANCH_NAME == 'master'
isDevelop = BRANCH_NAME == 'develop'
isHotfix  = BRANCH_NAME.startsWith('hotfix')
isRelease = BRANCH_NAME.startsWith('release')

if (isMaster || isDevelop) {
  defaults.action_type.add('start_release')
  defaults.protect_branch = true
  if (isMaster) {
    defaults.release_type = 'hotfix'
  } else if (isDevelop) {
    defaults.release_type = 'release'
  }
  defaults.action_help = " (start for ${defaults.release_type} on current branch)"
} else if (isHotfix || isRelease) {
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
      defaultValue: '0.0.0'
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
          if (params.wipe) {
            deleteDir()
            checkout scm
          }
          currentBuild.displayName += ' - ' + params.action_type
          // utils = load 'utils.groovy'
          currentBranch = BRANCH_NAME
          baseBranches = []
          // reposList = utils.getReposList()
          stats = [
            repos: [:],
            list: '',
            success: 0,
            total: 0
          ]
          reposList.each {
            stats.repos.put(it.owner + '/' + it.name, [:])
          }

          if (params.action_type == 'print_branches') {

            Boolean printed
            stats.repos.each { repo, status ->
              printed = printBranches(repo)
              status.primary = (printed) ? 'success' : 'failure'
              echo "${repo}: print - ${status.primary}"
            }

          } else if (params.action_type == 'start_release') {

            currentBranch = defaults.release_type + '/v' + params.vesion
            baseBranches = [BRANCH_NAME]
            Boolean created
            Boolean locked

            stats.repos.each { repo, status ->
              if (checkRemoteBranch(repo, currentBranch)) {
                echo "Branch already ${currentBranch} exists."
                status.primary = 'skip'
              } else {
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  if (!checkRemoteBranch(repo, 'develop')
                    && !createBranch(repo, 'develop', 'master'))
                    error("Can't create develop branch.")

                  created = createBranch(repo, currentBranch, baseBranches[0])
                  status.primary = (created) ? 'success' : 'failure'
                }
              }
              echo "${repo}: create - ${status.primary}"

              if (params.protect_branch) {
                locked = protectBranch(repo, currentBranch)
                status.secondary = (locked) ? 'lock' : ''
                echo "${repo}: ${status.secondary}"
              }
            }

          } else if (params.action_type == 'merge_release') {

            baseBranches = ['master']
            Boolean merged

            stats.repos.each { repo, status ->
              if (!checkRemoteBranch(repo, currentBranch)) {
                echo "Branch doesn't ${currentBranch} exist."
                status.primary = 'skip'
              } else {
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  merged = mergeBranch(repo, currentBranch, baseBranches)
                  status.primary = (merged) ? 'success' : 'failure'
                }
              }
              echo "${repo}: merge - ${status.primary}"
            }

          } else if (params.action_type == 'finish_release') {

            baseBranches = ['master', 'develop']
            if (!params.extra_branch.isEmpty())
              baseBranches.add(params.extra_branch)
            Boolean unlocked
            Boolean merged
            Boolean deleted

            stats.repos.each { repo, status ->
              if (!checkRemoteBranch(repo, currentBranch)) {
                echo "Branch doesn't ${currentBranch} exist."
                status.primary = 'skip'
              } else {
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  merged = mergeBranch(repo, currentBranch, baseBranches)
                  status.primary = (merged) ? 'success' : 'failure'
                  echo "${repo}: merge - ${status.primary}"
                  if (merged) {
                    unprotectBranch(repo, currentBranch)
                    deleted = deleteBranch(repo, currentBranch)
                    status.secondary = (deleted) ? 'delete' : ''
                    echo "${repo}: ${status.secondary}"
                  }
                }
              }
            }

          } else if (params.action_type == 'unprotect_release') {

            Boolean unlock
            stats.repos.each { repo, status ->
              unlock = unprotectBranch(repo, currentBranch)
              status.primary = (unlock) ? 'success' : 'failure'
            }

          }
        }
      }
    }
  }
  post {
    success {
      script {
        stats.success = stats.repos.findAll { repo, status ->
          status.primary == 'skip' || status.primary == 'success'
        }.size()
        stats.total = stats.repos.size()

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
  Integer ret = sh (
    label: "${repo}: check branch ${branch}",
    script: "git ls-remote --exit-code git@github.com:${repo}.git ${branch}",
    returnStatus: true
  )
  println "ret: $ret"
  return ret == 0
}

def createBranch(String repo, String branch, String baseBranch) {
  Integer ret = sh (
    label: "${repo}: start ${branch}",
    script: """
      git checkout ${baseBranch}
      git reset --hard origin/${baseBranch}
      git checkout -B ${branch}
      git push origin ${branch}
      git branch -vv
    """,
    returnStatus: true
  )
  println "ret: $ret"
  return ret == 0
}

def mergeBranch(String repo, String branch, ArrayList baseBranches) {
  Integer ret = sh (
    label: "${repo}: merge ${branch} into ${baseBranches.join(' ')}",
    script: """#!/bin/bash -xe
      git checkout ${branch}
      echo \$?
      git reset --hard origin/${branch}
      echo \$?
      base_branches=(${baseBranches.join(' ')})
      merged=0
      rev_branch=\$(git rev-parse @)
      echo \$?
      for base in "\${base_branches[@]}"; do
        git checkout \$base
        echo \$?
        git reset --hard origin/\$base
        echo \$?
        rev_base=\$(git rev-parse @)
        echo \$?
        if [[ \$rev_branch == \$rev_base ]]; then
          ((++merged))
          echo "No new commits."
          continue
        fi
        gh pr create --repo ${repo} --base \$base --head ${branch} \
          --title "Merge branch ${branch} into \$base" --fill
        echo \$?
        git merge ${branch} --no-edit --no-ff \
          -m "Merge branch ${branch} into \$base" || \
        continue
        git push origin \$base
        echo \$?
        ((++merged))
      done
      git branch -vv
      echo \$?
      [[ \$merged -ne \${#base_branches[@]} ]] && exit 2
      echo \$?
      exit 0
    """,
    returnStatus: true
  )
  println "ret: $ret"
  return ret == 0
}

def deleteBranch(String repo, String branch) {
  Integer ret = sh (
    label: "${repo}: delete ${branch}",
    script: """
      git branch -D ${branch}
      git push --delete origin ${branch}
    """,
    returnStatus: true
  )
  println "ret: $ret"
  return ret == 0
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
  Integer ret = sh (
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
  )
  println "ret: $ret"
  return ret == 0
}

def unprotectBranch(String repo, String branch) {
  Integer ret = sh (
    label: "${repo}: unprotect ${branch}",
    script: "gh api -X DELETE repos/${repo}/branches/${branch}/protection",
    returnStatus: true
  )
  println "ret: $ret"
  return ret == 0
}

def sendNotification() {
  Integer chatId = -579429080 // test
  String text = ''
  switch(params.action_type) {
    case 'start_release':
      text = "Branch `${currentBranch}` created from `${baseBranches[0]}`"
      break
    case 'merge_release':
      text = "Branch `${currentBranch}` merged into `${baseBranches[0]}`"
      break
    case 'finish_release':
      text = "Branch `${currentBranch}` merged into "
      text += baseBranches.collect({"`$it`"}).join(', ')
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

  if ((params.action_type == 'start_release'
    || params.action_type == 'merge_release'
    || params.action_type == 'finish_release')
    && params.notify && stats.success > 0) {
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
