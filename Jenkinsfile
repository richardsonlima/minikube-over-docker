node('docker') {
  properties([disableConcurrentBuilds()])

  wkdir = "src/istio.io/istio"

  stage('Checkout') {
    checkout scm
  }

  // withRegistry writes to /home/ubuntu/.dockercfg outside of the container
  // (even if you run it inside the docker plugin) which won't be visible
  // inside the builder container, so copy them somewhere that will be
  // visible.  We will symlink to .dockercfg only when needed to reduce
  // the chance of accidentally using the credentials outside of push
  docker.withRegistry('https://quay.io', 'name-of-your-credentials-in-jenkins') {
    stage('Load Push Credentials') {
      sh "cp ~/.dockercfg ${pwd()}/.dockercfg-quay-creds"
    }
  }

  k8sImage = docker.build(
    "k8s-${env.BUILD_TAG}",
    "-f $wkdir/.jenkins/Dockerfile.minikube " +
    "$wkdir/.jenkins/"
  )
  k8sImage.withRun('--privileged') { k8s ->
    stage('Get kubeconfig') {
      sh "docker exec ${k8s.id} /bin/bash -c \"while ! [ -e /kubeconfig ]; do echo waiting for kubeconfig; sleep 3; done\""
      sh "rm -f ${pwd()}/kubeconfig && docker cp ${k8s.id}:/kubeconfig ${pwd()}/kubeconfig"

      // Replace "127.0.0.1" with the path that peer containers can use to
      // get to minikube.
      // minikube will bake certs including the subject "kubernetes" so
      // the kube-api server needs to be reachable from the client's concept
      // of "https://kubernetes:8443" or kubectl will refuse to connect. 
      sh "sed -i'' -e 's;server: https://127.0.0.1:8443;server: https://kubernetes:8443;' kubeconfig"
    }

    builder = docker.build(
      "istio-builder-${env.BUILD_TAG}",
      "-f $wkdir/.jenkins/Dockerfile.jenkins-build " +
        "--build-arg UID=`id -u` --build-arg GID=`id -g` " +
        "$wkdir/.jenkins",
    )

    builder.inside(
      "-e GOPATH=${pwd()} " +
      "-e HOME=${pwd()} " +
      "-e PATH=${pwd()}/bin:\$PATH " +
      "-e KUBECONFIG=${pwd()}/kubeconfig " +
      "-e DOCKER_HOST=\"tcp://kubernetes:2375\" " +
      "--link ${k8s.id}:kubernetes"
    ) {
      stage('Check') {
        sh "ls -al"

        // If there are old credentials from a previous build, destroy them -
        // we will only load them when needed in the push stage
        sh "rm -f ~/.dockercfg"

        sh "cd $wkdir && go get -u github.com/golang/lint/golint"
        sh "cd $wkdir && make check"
      }

      stage('Build') {
        sh "cd $wkdir && make depend"
        sh "cd $wkdir && make build"
      }

      stage('Test') {
        sh "cp kubeconfig $wkdir/pilot/platform/kube/config"
        sh """PROXYVERSION=\$(grep envoy-debug $wkdir/pilot/docker/Dockerfile.proxy_debug  |cut -d: -f2) &&
          PROXY=debug-\$PROXYVERSION &&
          curl -Lo - https://storage.googleapis.com/istio-build/proxy/envoy-\$PROXY.tar.gz | tar xz &&
          mv usr/local/bin/envoy ${pwd()}/bin/envoy &&
          rm -r usr/"""
        sh "cd $wkdir && make test"
      }

      stage('Push') {
        sh "cd && ln -sf .dockercfg-quay-creds .dockercfg"
        sh "cd $wkdir && " +
          "make HUB=yourhub TAG=$BUILD_TAG push"
        gitTag = getTag(wkdir)
        if (gitTag) {
          sh "cd $wkdir && " +
            "make HUB=yourhub TAG=$gitTag push"
        }
        sh "cd && rm .dockercfg"
      }
    }
  }
}

String getTag(String wkdir) {
  return sh(
    script: "cd $wkdir && " +
      "git describe --exact-match --tags \$GIT_COMMIT || true",
    returnStdout: true
  ).trim()
}

