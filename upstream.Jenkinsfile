#!/usr/bin/env groovy

// PARAMETERS for this pipeline:
// def FORCE_BUILD = "false"

def SOURCE_BRANCH = "crw-2.0" // or master :: branch of source repo from which to find and sync commits to pkgs.devel repo
def SOURCE_REPO = "eclipse/che-operator" //source repo from which to find and sync commits to pkgs.devel repo
def GIT_PATH = "containers/codeready-workspaces-operator" // dist-git repo to use as target

def GIT_BRANCH = "crw-2.0-rhel-8" // target branch in dist-git repo, eg., crw-2.0-rhel-8
def SCRATCH = "false"
def PUSH_TO_QUAY = "true"
def QUAY_PROJECT = "operator" // also used for the Brew dockerfile params

def OLD_SHA=""

def buildNode = "rhel7-releng" // slave label
timeout(120) {
	node("${buildNode}"){ stage "Sync repos"
		cleanWs()
	        withCredentials([
                file(credentialsId: 'crw-build.keytab', variable: 'CRW_KEYTAB'),
                usernamePassword(credentialsId: 'devstudio-release', passwordVariable: 'SOURCE_GIT_PASSWORD', usernameVariable: 'SOURCE_GIT_USERNAME')]
            ) {
		      checkout([$class: 'GitSCM',
		        branches: [[name: "${SOURCE_BRANCH}"]],
		        doGenerateSubmoduleConfigurations: false,
		        credentialsId: 'devstudio-release',
		        poll: true,
		        extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: "sources"]],
		        submoduleCfg: [],
		        userRemoteConfigs: [[url: "https://${SOURCE_GIT_USERNAME}@${SOURCE_GIT_PASSWORD}@github.com/${SOURCE_REPO}.git"]]])

  		      def BOOTSTRAP = '''#!/bin/bash -xe

# bootstrapping: if keytab is lost, upload to
# https://codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com/credentials/store/system/domain/_/
# then set Use secret text above and set Bindings > Variable (path to the file) as ''' + CRW_KEYTAB + '''
chmod 700 ''' + CRW_KEYTAB + ''' && chown ''' + USER + ''' ''' + CRW_KEYTAB + '''
# create .k5login file
echo "crw-build/codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com@REDHAT.COM" > ~/.k5login
chmod 644 ~/.k5login && chown ''' + USER + ''' ~/.k5login
 echo "pkgs.devel.redhat.com,10.16.101.66 ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAplqWKs26qsoaTxvWn3DFcdbiBxqRLhFngGiMYhbudnAj4li9/VwAJqLm1M6YfjOoJrj9dlmuXhNzkSzvyoQODaRgsjCG5FaRjuN8CSM/y+glgCYsWX1HFZSnAasLDuW0ifNLPR2RBkmWx61QKq+TxFDjASBbBywtupJcCsA5ktkjLILS+1eWndPJeSUJiOtzhoN8KIigkYveHSetnxauxv1abqwQTk5PmxRgRt20kZEFSRqZOJUlcl85sZYzNC/G7mneptJtHlcNrPgImuOdus5CW+7W49Z/1xqqWI/iRjwipgEMGusPMlSzdxDX4JzIx6R53pDpAwSAQVGDz4F9eQ==
" >> ~/.ssh/known_hosts

ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts

# see https://mojo.redhat.com/docs/DOC-1071739
if [[ -f ~/.ssh/config ]]; then mv -f ~/.ssh/config{,.BAK}; fi
echo "
GSSAPIAuthentication yes
GSSAPIDelegateCredentials yes

Host pkgs.devel.redhat.com
User crw-build/codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com@REDHAT.COM
" > ~/.ssh/config
chmod 600 ~/.ssh/config

# initialize kerberos
export KRB5CCNAME=/var/tmp/crw-build_ccache
kinit "crw-build/codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com@REDHAT.COM" -kt ''' + CRW_KEYTAB + '''
klist # verify working

hasChanged=0

# TODO rename upstream Dockerfile to rhel.Dockerfile in che-operator repo
SOURCEDOCKERFILE=${WORKSPACE}/sources/Dockerfile
if [[ -f ${WORKSPACE}/sources/rhel.Dockerfile ]]; then
  SOURCEDOCKERFILE=${WORKSPACE}/sources/rhel.Dockerfile
fi

# REQUIRE: skopeo
# TODO merge changes in generatePRs branch to master when ready
curl -L -s -S https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/generatePRs/product/updateBaseImages.sh -o /tmp/updateBaseImages.sh
chmod +x /tmp/updateBaseImages.sh
cd ${WORKSPACE}/sources
  git checkout --track origin/''' + SOURCE_BRANCH + ''' || true
  git config user.email nickboldt+devstudio-release@gmail.com
  git config user.name "devstudio-release"
  git config --global push.default matching

  # TODO verify we can generate a PR here; copy this to other Jenkinsfiles and test there too
  set +e
  /tmp/updateBaseImages.sh -b ''' + SOURCE_BRANCH + ''' -f ${SOURCEDOCKERFILE##*/} -maxdepth 1 --pull-request
  set -e
cd ..

# fetch sources to be updated
GIT_PATH="''' + GIT_PATH + '''"
if [[ ! -d ${WORKSPACE}/target ]]; then git clone ssh://crw-build@pkgs.devel.redhat.com/${GIT_PATH} target; fi
cd ${WORKSPACE}/target
git checkout --track origin/''' + GIT_BRANCH + ''' || true
git config user.email crw-build@REDHAT.COM
git config user.name "CRW Build"
git config --global push.default matching
cd ..

'''
		      sh BOOTSTRAP

		      OLD_SHA = sh(script: '''#!/bin/bash -xe
		      cd ${WORKSPACE}/target; git rev-parse HEAD
		      ''', returnStdout: true)
		      println "Got OLD_SHA in target folder: " + OLD_SHA

		      sh BOOTSTRAP + '''

# rsync files in github to dist-git
SYNC_FILES=".gitignore Dockerfile Gopkg.lock Gopkg.toml README.md  cmd templates e2e pkg vendor version"
for d in ${SYNC_FILES}; do
  if [[ -f ${WORKSPACE}/sources/${d} ]]; then
    rsync -zrlt ${WORKSPACE}/sources/${d} ${WORKSPACE}/target/${d}
  elif [[ -d ${WORKSPACE}/sources/${d} ]]; then
    # copy over the files
    rsync -zrlt ${WORKSPACE}/sources/${d}/* ${WORKSPACE}/target/${d}/
    # sync the directory and delete from target if deleted from source
    rsync -zrlt --delete ${WORKSPACE}/sources/${d}/ ${WORKSPACE}/target/${d}/
  fi
done

cp -f ${SOURCEDOCKERFILE} ${WORKSPACE}/target/Dockerfile

# TODO should this be a branch instead of just master?
CRW_VERSION=`wget -qO- https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/master/dependencies/VERSION`
#apply patches
sed -i ${WORKSPACE}/target/Dockerfile \
  -e "s#FROM registry.redhat.io/#FROM #g" \
  -e "s#FROM registry.access.redhat.com/#FROM #g" \
  -e "s/# *RUN yum /RUN yum /g" \
  -e "s@# *ADD controller-manifests /manifests@ADD controller-manifests /manifests@g"

METADATA='ENV SUMMARY="Red Hat CodeReady Workspaces ''' + QUAY_PROJECT + ''' container" \\\r
    DESCRIPTION="Red Hat CodeReady Workspaces ''' + QUAY_PROJECT + ''' container" \\\r
    PRODNAME="codeready-workspaces" \\\r
    COMPNAME="''' + QUAY_PROJECT + '''-rhel8" \r
LABEL summary="$SUMMARY" \\\r
      description="$DESCRIPTION" \\\r
      io.k8s.description="$DESCRIPTION" \\\r
      io.k8s.display-name=\"$DESCRIPTION" \\\r
      io.openshift.tags="$PRODNAME,$COMPNAME" \\\r
      com.redhat.component="$PRODNAME-$COMPNAME-container" \\\r
      name="$PRODNAME/$COMPNAME" \\\r
      version="'$CRW_VERSION'" \\\r
      license="EPLv2" \\\r
      maintainer="Nick Boldt <nboldt@redhat.com>" \\\r
      io.openshift.expose-services="" \\\r
      com.redhat.delivery.appregistry="true" \\\r
      usage="" \r'

echo -e "$METADATA" >> ${WORKSPACE}/target/Dockerfile

# push changes in github to dist-git
cd ${WORKSPACE}/target
if [[ \$(git diff --name-only) ]]; then # file changed
	OLD_SHA=\$(git rev-parse HEAD) # echo ${OLD_SHA:0:8}
	git add ${SYNC_FILES}
    /tmp/updateBaseImages.sh -b ''' + GIT_BRANCH + ''' --nocommit
	git commit -s -m "[sync] Update from ''' + SOURCE_REPO + ''' @ ${SOURCE_SHA:0:8}" ${SYNC_FILES}
	git push origin ''' + GIT_BRANCH + '''
	NEW_SHA=\$(git rev-parse HEAD) # echo ${NEW_SHA:0:8}
	if [[ "${OLD_SHA}" != "${NEW_SHA}" ]]; then hasChanged=1; fi
	echo "[sync] Updated pkgs.devel @ ${NEW_SHA:0:8} from ''' + SOURCE_REPO + ''' @ ${SOURCE_SHA:0:8}"
else
    # file not changed, but check if base image needs an update
    # (this avoids having 2 commits for every change)
    cd ${WORKSPACE}/target
    OLD_SHA=\$(git rev-parse HEAD) # echo ${OLD_SHA:0:8}
    /tmp/updateBaseImages.sh -b ''' + GIT_BRANCH + '''
    NEW_SHA=\$(git rev-parse HEAD) # echo ${NEW_SHA:0:8}
    if [[ "${OLD_SHA}" != "${NEW_SHA}" ]]; then hasChanged=1; fi
    cd ..
fi
cd ..

if [[ ''' + FORCE_BUILD + ''' == "true" ]]; then hasChanged=1; fi
if [[ ${hasChanged} -eq 1 ]]; then
  for QRP in ''' + QUAY_PROJECT + '''; do
    QUAY_REPO_PATH=""; if [[ ''' + PUSH_TO_QUAY + ''' == "true" ]]; then QUAY_REPO_PATH="${QRP}-rhel8"; fi
    curl \
"https://codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com/job/get-sources-rhpkg-container-build/buildWithParameters?\
token=CI_BUILD&\
cause=${QUAY_REPO_PATH}+respin+by+${BUILD_TAG}&\
GIT_BRANCH=''' + GIT_BRANCH + '''&\
GIT_PATHs=containers/codeready-workspaces-${QRP}&\
QUAY_REPO_PATHs=${QUAY_REPO_PATH}&\
JOB_BRANCH=master&\
FORCE_BUILD=true&\
SCRATCH=''' + SCRATCH + '''"
  done
fi

if [[ ${hasChanged} -eq 0 ]]; then
  echo "No changes upstream, nothing to commit"
fi
'''
    		}
	        def NEW_SHA = sh(script: '''#!/bin/bash -xe
	        cd ${WORKSPACE}/target; git rev-parse HEAD
	        ''', returnStdout: true)
	        println "Got NEW_SHA in target folder: " + NEW_SHA

	        if (NEW_SHA.equals(OLD_SHA)) {
	          currentBuild.result='UNSTABLE'
	        }
	}
}