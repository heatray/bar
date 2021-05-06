defaults = [
  action_type: [ 'none', 'print_branches' ],
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
          branch = defaults.release_type + '/v' + params.vesion
          stats = [
            list: '',
            success: 0,
            total: 0
          ]
          notifyMessage = ''
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
              checkoutRepos(BRANCH_NAME)
              startRelease(branch, BRANCH_NAME, params.protect_branch)
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
              checkoutRepos(BRANCH_NAME)
              mergeRelease(BRANCH_NAME)
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
              checkoutRepos(BRANCH_NAME)
              finishRelease(BRANCH_NAME, params.extra_branch)
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
              echo "Unprotect Release"
              // unprotectRelease(branch)
            }
          }
        }
      }
    }
  }
}

def checkoutRepo(Map repo, String branch = 'master') {
  env.REPO = repo.owner + '/' + repo.name
  env.BRANCH = branch
  sh (
    label: "${REPO}: checkout",
    script: '''#!/bin/bash -xe
      if [[ ! -d $REPO ]]; then
        git clone -b $BRANCH git@github.com:$REPO.git $REPO
      else
        if [[ $(git rev-parse --is-inside-work-tree) != 'true' ]]; then
          rm -rfv $REPO
          git clone -b $BRANCH git@github.com:$REPO.git $REPO
        else
          pushd $REPO
          git fetch -p
          if [ $(git branch -a | grep $BRANCH | wc -c) -eq 0 ]; then
            exit 0
          fi
          git reset --hard origin/$BRANCH
          git pull -f origin $BRANCH
          popd      
        fi
      fi
    '''
  )
}

def checkoutRepos(String branch = 'master') {    
  // for (repo in utils.getReposList()) {
  for (repo in reposList) {
    checkoutRepo(repo, branch)
  }
}

def createBranch(Map repo, String branch, String baseBranch) {
  env.REPO = repo.owner + '/' + repo.name
  env.BRANCH = branch
  env.BASE_BRANCH = baseBranch
  return sh (
    label: "${REPO}: start ${BRANCH}",
    script: '''#!/bin/bash -xe
      # create develop if doesn't exist
      if ! git ls-remote --refs --exit-code . origin/develop; then
        git checkout -f master
        git checkout -b develop
        git push origin develop
        git checkout -f $BASE_BRANCH
      fi
      if git ls-remote --refs --exit-code . origin/$BRANCH; then
        exit 0
      fi
      git checkout -B $BRANCH
      git push origin $BRANCH
    ''',
    returnStatus: true
  )
}

def mergeBranch(Map repo, String branch, ArrayList baseBranches) {
  env.REPO = repo.owner + '/' + repo.name
  env.BRANCH = branch
  env.BASE_BRANCHES = baseBranches.join(' ')
  env.BASE_BRANCHES_SIZE = baseBranches.size()
  return sh (
    label: "${REPO}: merge ${BRANCH} into ${BASE_BRANCHES}",
    script: '''#!/bin/bash -xe
      if [ $(git branch -a | grep $BRANCH | wc -c) -eq 0 ]; then
        exit 0
      fi
      merged=0
      for base in $BASE_BRANCHES; do
        git checkout -f $BRANCH
        git pull --ff-only origin $BRANCH
        gh pr create \
          --repo $REPO \
          --base $base \
          --head $BRANCH \
          --title "Merge branch $BRANCH into $base" \
          --fill || \
        true
        git checkout $base
        git pull --ff-only origin $base
        git merge $BRANCH \
          --no-edit --no-ff \
          -m "Merge branch $BRANCH into $base" || \
        continue
        git push origin $base
        ((++merged))
      done
      if [ $merged -ne $BASE_BRANCHES_SIZE ]; then
        exit 2
      fi
    ''',
    returnStatus: true
  )
}

def deleteBranch(Map repo, String branch) {
  env.REPO = repo.owner + '/' + repo.name
  env.BRANCH = branch
  return sh (
    label: "${REPO}: delete ${BRANCH}",
    script: '''#!/bin/bash -xe
      if [ $(git branch -a | grep $BRANCH | wc -c) -eq 0 ]; then
        exit 0
      fi
      git branch -D $BRANCH
      git push --delete origin $BRANCH
    ''',
    returnStatus: true
  )
}

def protectBranch(Map repo, String branch) {
  env.REPO = repo.owner + '/' + repo.name
  env.BRANCH = branch
  return sh (
    label: "${REPO}: protect ${BRANCH}",
    script: '''#!/bin/bash -xe
      echo '{
        "required_status_checks": null,
        "enforce_admins": true,
        "required_pull_request_reviews": null,
        "restrictions": {
          "users": [],
          "teams": []
        }
      }' | \
      gh api -X PUT repos/$REPO/branches/$BRANCH/protection --input -
    ''',
    returnStatus: true
  )
}

def unprotectBranch(Map repo, String branch) {
  env.REPO = repo.owner + '/' + repo.name
  env.BRANCH = branch
  return sh (
    label: "${REPO}: unprotect ${BRANCH}",
    script: 'gh api -X DELETE repos/$REPO/branches/$BRANCH/protection',
    returnStatus: true
  )
}

def startRelease(String branch, String baseBranch, Boolean protect) {
  stats.success = 0
  stats.total = reposList.size()
  for (repo in reposList) {
    dir (repo.owner + '/' + repo.name) {
      Integer retC = createBranch(repo, branch, baseBranch)
      if (!protect) {
        fillStats(retC == 0)
      } else {
        Integer retP = protectBranch(repo, branch)
        fillStats(retC == 0, retP == 0)
      }
    }
  }
  setBuildStatus()
  notifyMessage = "Branch `${branch}` created from `${baseBranch}`"
  notifyMessage += " \\[${stats.success}/${stats.total}]\n"
  notifyMessage += stats.list
  if (stats.success > 0) sendNotification()
}

def mergeRelease(String branch)
{
  stats.success = 0
  stats.total = reposList.size()
  for (repo in reposList) {
    dir (repo.owner + '/' + repo.name) {
      Integer retM = mergeBranch(repo, branch, ['master'])
      fillStats(retM == 0)
    }
  }
  setBuildStatus()
  notifyMessage = "Branch `${branch}` merged into `master`"
  notifyMessage += " \\[${stats.success}/${stats.total}]\n"
  notifyMessage += stats.list
  if (stats.success > 0) sendNotification()
}

def finishRelease(String branch, String extraBranch = '')
{
  stats.success = 0
  stats.total = reposList.size()
  ArrayList baseBranches = ['master', 'develop']
  if (!extraBranch.isEmpty()) baseBranches.add(extraBranch)
  for (repo in reposList) {
    dir (repo.owner + '/' + repo.name) {
      Integer retM = mergeBranch(repo, branch, baseBranches)
      if (retM != 0) {
        fillStats(retM == 0)
      } else if (retM == 0) {
        Integer retP = unprotectBranch(repo, branch)
        Integer retD = deleteBranch(repo, branch)
        fillStats(retD == 0, retP != 0)
      }
    }
  }
  setBuildStatus()
  String tgBranches = baseBranches.collect({"`$it`"}).join(', ')
  notifyMessage = "Branch `${branch}` merged into ${tgBranches}"
  notifyMessage += " \\[${stats.success}/${stats.total}]\n"
  notifyMessage += stats.list
  if (stats.success > 0) sendNotification()
}

def fillStats(Boolean action, Boolean protect = false) {
  if (action) stats.success++
  if (action) stats.list += '✅' else stats.list += '❎'
  if (protect) stats.list += '🛡'
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
  withCredentials([string(
    credentialsId: 'telegram-bot-token',
    variable: 'TELEGRAM_TOKEN'
  )]) {
    sh "curl -X POST -s -S \
      -d parse_mode=markdown \
      -d chat_id=${chatId} \
      --data-urlencode text='${notifyMessage}' \
      https://api.telegram.org/bot\$TELEGRAM_TOKEN/sendMessage"
  }
}
