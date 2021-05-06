defaults = [
  action_type: [ 'none', 'print_branches' ]
]

isMaster  = BRANCH_NAME == 'master'
isDevelop = BRANCH_NAME == 'develop'
isHotfix  = BRANCH_NAME.startsWith('hotfix')
isRelease = BRANCH_NAME.startsWith('release')

if (BRANCH_NAME == 'master') {
  defaults.release_type = 'hotfix'
} else if (BRANCH_NAME == 'develop') {
  defaults.release_type = 'release'
}

if (BRANCH_NAME == 'master' || BRANCH_NAME == 'develop') {
  defaults.action_type.add('start_release')
}

if (BRANCH_NAME.startsWith('hotfix') || BRANCH_NAME.startsWith('release')) {
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
            list: '',
            success: 0,
            total: 0
          ]
          notifyMessage = ''
        }
      }
    }
    stage('Flow') {
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
              mergeRelease(branch)
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
              // if (!params.extra_branch.isEmpty()) {
              //   finishRelease(branch, params.extra_branch)
              // } else {
              //   finishRelease(branch)
              // }
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
  return sh (
    label: "${REPO}: merge ${BRANCH} into ${BASE_BRANCHES}",
    script: '''#!/bin/bash -xe
      if [ $(git branch -a | grep '$BRANCH' | wc -c) -eq 0 ]; then
        exit 0
      fi
      merged=0
      for base in ${BASE_BRANCHES[*]}; do
        git checkout -f $BRANCH
        git pull --ff-only origin $BRANCH
        gh pr create \
          --repo $REPO \
          --base $base \
          --head $BRANCH \
          --title "Merge branch $BRANCH into \$base\" \
          --fill || \
        true
        git checkout $base
        git pull --ff-only origin $base
        git merge $BRANCH \
          --no-edit --no-ff \
          -m \"Merge branch $BRANCH into $base\" || \
        continue
        git push origin $base
        ((++merged))
      done
      if [ $merged -ne ${#BASE_BRANCHES[@]} ]; then
        exit 2
      fi
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
      if (!protect) {
        setStats(retC == 0)
      } else {
        Integer retP = protectBranch(repo, branch)
        setStats(retC == 0, retP == 0)
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
    dir (repo.dir) {
      Integer retM = mergeBranch(repo, branch, ['master'])
      setStats(retM == 0)
    }
  }
  setBuildStatus()
  notifyMessage = "Branch `${branch}` merged into `master`"
  notifyMessage += " \\[${stats.success}/${stats.total}]\n"
  notifyMessage += stats.list
  if (stats.success > 0) sendNotification()
}

def setStats(Boolean action, Boolean protect = false) {
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
