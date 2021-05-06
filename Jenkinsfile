defaults = [
  action_type: [
    'none',
    'start_release',
    'merge_release',
    'finish_release',
    'print_branches',
    'unprotect_release'
  ]
]

isMaster  = BRANCH_NAME == 'master'
isDevelop = BRANCH_NAME == 'develop'
isHotfix  = BRANCH_NAME.startsWith('hotfix')
isRelease = BRANCH_NAME.startsWith('release')

if (BRANCH_NAME == 'master') {
  defaults.action_type.remove(2) // merge_release
  defaults.action_type.remove(2) // finish_release
  defaults.action_type.remove(3) // unprotect_release
  defaults.release_type = 'hotfix'
}

if (BRANCH_NAME == 'develop') {
  defaults.action_type.remove(2) // merge_release
  defaults.action_type.remove(2) // finish_release
  defaults.action_type.remove(3) // unprotect_release
  defaults.release_type = 'release'
}

if (BRANCH_NAME.startsWith('hotfix')) {
  defaults.action_type.remove(1) // start_release
  // defaults.base_branch = 'master'
}

if (BRANCH_NAME.startsWith('release')) {
  defaults.action_type.remove(1) // start_release
  // defaults.base_branch = 'develop'
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
    // choice (
    //   name:         'release_type',
    //   description:  'Release type',
    //   choices:      ['hotfix', 'release']
    // )
    string (
      name:         'vesion',
      description:  'Release version',
      defaultValue: '0.0.0'
    )
    booleanParam (
      name:         'protect_branch',
      description:  'Protect branch (for start_release only)',
      defaultValue: true
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
            success: 0,
            total: 0,
            list: ''
          ]
          notifyMessage = ''
        }
      }
    }
    stage('Flow') {
      parallel {
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
            expression { params.action_type == 'finish_release' }
            beforeAgent true
          }
          steps {
            script {
              checkoutRepos(BRANCH_NAME)
              // mergeRelease(branch)
            }
          }
        }
        stage('Finish Release') {
          when {
            expression { params.action_type == 'merge_release' }
            beforeAgent true
          }
          steps {
            script {
              checkoutRepos(BRANCH_NAME)
              // if (!params.extra_branch.isEmpty()) {
              //   finishRelease(branch, params.extra_branch)
              // } else {
              //   finishRelease(branch)
              // }
            }
          }
        }
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
  sh label: "${REPO}: checkout",
    script: '''#!/bin/bash -xe
      if [[ ! -d $REPO ]]; then
        git clone -b $BRANCH git@github.com:$REPO.git $REPO
      else
        if [[ $(git rev-parse --is-inside-work-tree) != 'true' ]]; then
          rm -rfv $REPO
          git clone -b $BRANCH git@github.com:$REPO.git $REPO
        else
          pushd $REPO
          git fetch -p origin $BRANCH
          git reset --hard origin/$BRANCH
          git pull -f origin $BRANCH
          #git clean -xdf
          popd      
        fi
      fi
    '''
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
      if ! git ls-remote --heads --exit-code . develop; then
        git checkout -f master
        git checkout -b develop
        git push origin develop
        git checkout -f $BASE_BRANCH
      fi
      if git ls-remote --heads --exit-code . $BRANCH; then
        exit 0
      fi
      git checkout -B $BRANCH
      git push origin $BRANCH
      sleep 3s
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

def startRelease(String branch, String baseBranch, Boolean protect) {
  stats.success = 0
  stats.total = reposList.size()
  for (repo in reposList) {
    dir (repo.owner + '/' + repo.name) {
      Integer retC = createBranch(repo, branch, baseBranch)
      if (protect) {
        Integer retP = protectBranch(repo, branch)
        regStat(retC == 0 && retP == 0)
      } else {
        regStat(retC == 0)
      }
    }
  }
  setBuildStatus()
  notifyMessage = "Branch `${branch}` created from `${baseBranch}`"
  notifyMessage += " \\[${stats.success}/${stats.total}]\n"
  notifyMessage += stats.list
  if (stats.success > 0) sendNotification()
}

def regStat(Boolean ret) {
  if (ret) {
    stats.list += "✅"
    stats.success++
  } else {
    stats.list += "❎"
  }
  stats.list += " ${repo}\n"
}

def setBuildStatus() {
  if (stats.success == 0) {
    currentBuild.result = "FAILURE"
  } else if (stats.success != stats.total) {
    currentBuild.result = "UNSTABLE"
  } else if (stats.success == stats.total) {
    currentBuild.result = "SUCCESS"
  }
}

def sendNotification() {
  int chatId = -579429080 // test

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
