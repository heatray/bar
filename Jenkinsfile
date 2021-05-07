defaults = [
  action_type: [ 'print_branches' ],
  version: '',
  protect_branch: false
]

isMaster  = BRANCH_NAME == 'master'
isDevelop = BRANCH_NAME == 'develop'
isHotfix  = BRANCH_NAME.startsWith('hotfix')
isRelease = BRANCH_NAME.startsWith('release')

if (isMaster) {
  defaults.release_type = 'hotfix'
} else if (isDevelop) {
  defaults.release_type = 'release'
}

if (isMaster || isDevelop) {
  defaults.action_type.add('start_release')
  defaults.version = '0.0.0'
  defaults.protect_branch = true
} else if (isHotfix || isRelease) {
  defaults.action_type.add('merge_release')
  defaults.action_type.add('finish_release')
  defaults.action_type.add('unprotect_release')
}

reposList = [
  [ owner: 'heatray', name: 'foo', dir: 'foo' ],
  [ owner: 'heatray', name: 'bar', dir: 'bar' ]
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
          stats = [
            repo: '',
            branch: '',
            baseBranches: [],
            list: '',
            success: 0,
            total: reposList.size()
          ]
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
              echo "Print Branches"
              // printReposBranches()
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
              stats.branch = defaults.release_type + '/v' + params.vesion
              stats.baseBranches = [BRANCH_NAME]

              for (i in reposList) {
                String repo = i.owner + '/' + i.name
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  if (sh (
                    script: "git ls-remote --exit-code git@github.com:${repo}.git develop",
                    returnStatus: true
                  ) != 0) {
                    Integer retD = createBranch(repo, 'develop', 'master')
                    if (retD != 0) error("Can't create develop branch")
                  }
                  Integer retC = createBranch(repo, stats.branch, baseBranches[0])
                  if (!params.protect_branch) {
                    fillStats(repo, retC == 0)
                  } else {
                    Integer retP = protectBranch(repo, stats.branch)
                    fillStats(repo, retC == 0, retP == 0)
                  }
                }
              }

              setBuildStatus()
              sendNotification()
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
              stats.branch = BRANCH_NAME
              stats.baseBranches = ['master']

              for (i in reposList) {
                String repo = i.owner + '/' + i.name
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  Integer retM = mergeBranch(repo, stats.branch, stats.baseBranches)
                  fillStats(repo, retM == 0)
                }
              }

              setBuildStatus()
              sendNotification()
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
              stats.branch = BRANCH_NAME
              stats.baseBranches = ['master', 'develop']
              if (!extraBranch.isEmpty()) stats.baseBranches.add(extraBranch)

              for (i in reposList) {
                String repo = i.owner + '/' + i.name
                dir ('repos/' + repo) {
                  checkoutRepo(repo)
                  Integer retM = mergeBranch(repo, stats.branch, stats.baseBranches)
                  if (retM != 0) {
                    fillStats(repo, retM == 0)
                  } else if (retM == 0) {
                    Integer retP = unprotectBranch(repo, stats.branch)
                    Integer retD = deleteBranch(repo, stats.branch)
                    fillStats(repo, retD == 0, retP != 0)
                  }
                }
              }

              setBuildStatus()
              sendNotification()
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
              stats.branch = BRANCH_NAME

              for (i in reposList) {
                String repo = i.owner + '/' + i.name
                Integer ret = unprotectBranch(repo, stats.branch)
                fillStats(repo, ret == 0)
              }

              setBuildStatus()
              println stats.list
            }
          }
        }
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
        if ! git ls-remote --refs --exit-code . origin/${branch}; then
          echo "Remote branch ${branch} doesn't exist."
          exit 0
        fi
        git reset --hard origin/${branch}
        git branch -vv
      else
        rm -rfv ./*
        git clone -b ${branch} git@github.com:${repo}.git .
      fi
    """
  )
}

def createBranch(String repo, String branch, String baseBranch) {
  return sh (
    label: "${repo}: start ${branch}",
    script: """
      if git ls-remote --refs --exit-code . origin/${branch}; then
        echo "Remote branch ${branch} exist."
        exit 0
      fi
      git reset --hard origin/${baseBranch}
      git branch -vv
      git checkout -B ${branch}
      git push origin ${branch}
    """,
    returnStatus: true
  )
}

def mergeBranch(String repo, String branch, ArrayList baseBranches) {
  return sh (
    label: "${repo}: merge ${branch} into ${baseBranches.join(' ')}",
    script: """
      if ! git ls-remote --refs --exit-code . origin/${branch}; then
        echo "Remote branch ${branch} doesn't exist."
        exit 0
      fi
      base_branches=(${baseBranches.join(' ')})
      merged=0
      for base in \${base_branches[*]}; do
        git reset --hard origin/${branch}
        git branch -vv
        gh pr create \
          --repo ${repo} \
          --base \$base \
          --head ${branch} \
          --title "Merge branch ${branch} into \$base" \
          --fill || \
        true
        git checkout \$base
        git pull --ff-only origin \$base
        git merge ${branch} \
          --no-edit --no-ff \
          -m "Merge branch ${branch} into \$base" || \
        continue
        git push origin \$base
        ((++merged))
      done
      if [ \$merged -ne \${#base_branches[@]} ]; then
        exit 2
      fi
    """,
    returnStatus: true
  )
}

def deleteBranch(String repo, String branch) {
  return sh (
    label: "${repo}: delete ${branch}",
    script: """
      if ! git ls-remote --refs --exit-code . origin/${branch}; then
        echo "Remote branch ${branch} doesn't exist."
        exit 0
      fi
      git branch -D ${branch}
      git push --delete origin ${branch}
    """,
    returnStatus: true
  )
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
  )
}

def unprotectBranch(String repo, String branch) {
  return sh (
    label: "${repo}: unprotect ${branch}",
    script: "gh api -X DELETE repos/${repo}/branches/${branch}/protection",
    returnStatus: true
  )
}

def fillStats(String repo, Boolean primary, Boolean secondary = false) {
  if (primary) stats.success++
  if (primary) stats.list += '✅' else stats.list += '❎'
  if (secondary) stats.list += '🛡'
  stats.list += " ${repo}\n"
}

def setBuildStatus() {
  if (stats.success == 0) {
    currentBuild.result = 'FAILURE'
  } else if (stats.success != stats.total) {
    currentBuild.result = 'UNSTABLE'
  } else if (stats.success == stats.total) {
    currentBuild.result = 'SUCCESS'
  }
}

def sendNotification() {
  Integer chatId = -579429080 // test
  String text
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
  }
  text += " \\[${stats.success}/${stats.total}]\n${stats.list}"

  if (params.notify && stats.success > 0) {
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
}
