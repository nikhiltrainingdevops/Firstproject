
pipeline {
  agent any

  environment {
    project = "ds-lsa"
    region = "us-west-2"
    python_library = "common-libs"
    module_name = "ds-common-libs"
    python_version = "${python_version}"
    library_module = "${library_module}"
    library_subdir = "${library_subdir}"
    library_branch = "${library_branch}"
    build_dir = "build"
    VIRTUALENV_DIR = ".venv"
    bucket = "${bucket}"
    layer_s3_key = "${s3_key}"
    tag = "${tag}"
    layer_name="${project}-layer-${python_library}-${tag}-${environment}"
    ecr_repo_name = "${project}-${python_library}-${environment}"
    ECR = "${aws_account_id}.dkr.ecr.${region}.amazonaws.com"
    ecr_endpoint = "${ECR}/${ecr_repo_name}"
  }
  
  
  stages {
    stage('Unit tests') {
      steps {
        sh ''' 
        git_branch_name=$(echo $GIT_BRANCH | sed -e "s|origin/||g")
        if [[ "${git_branch_name}" != dev ]]; then
          echo "$new_tag"
          new_tag="${git_branch_name}-${tag}"
          echo "$new_tag"
          environment="dev"
        if [[ "${git_branch_name}" = reg ]]; then
          echo "$new_tag"
          new_tag="${git_branch_name}-${tag}"
          echo "$new_tag"
          environment="reg"
        else
          echo "$new_tag"
          new_tag="${tag}"
          environment="dev"
         
        
        

        echo "Initializing python"
        PYENV_ROOT="$HOME/.pyenv"
        PATH="$PYENV_ROOT/bin:$PATH"
        eval "$(pyenv init -)"
        python3 -m venv --clear build && source build/bin/activate
        python3 --version

        echo "Initializing pytest environment"
        pip3 install --upgrade pip && pip3 install pylint
        pip3 install -r ./${module_name}/requirements.txt -r ./${module_name}/python3/requirements.txt
        ls -lah && pwd
        echo "Starting linting"
        #cd ${module_name} 
        find . -name '*.py' | grep -v -E 'venv|tests|build|legacy_helpers' | xargs pylint --rcfile=".pylintrc" | tee -a lint_logs.txt
        find . -name '*.py' | grep -v -E 'venv|tests|build|legacy_helpers' | xargs pylint --rcfile=".pylintrc"
        
        echo "Copying logs to s3"
        aws s3 cp ./lint_logs.txt s3://${project}-${environment}/lint_logs/
        cd ${module_name} 
        echo "Starting unit tests"
        export PYTHONPATH=LeviDSTools
        cd python3 && ls -lah && pwd
        python3 -m pytest
        '''
      }
    }


    stage('Layer Build') {
      steps {
        dir(build_dir) {
          sh '''
          echo ${python_version} > .python-version
          PYENV_ROOT="$HOME/.pyenv"
          PATH="$PYENV_ROOT/bin:$PATH"
          eval "$(pyenv init -)"
          python3 -m venv --clear build && source build/bin/activate
          python3 --version

          pip3 install --upgrade pip
          pip3 install \
            "git+ssh://git@github.levi-site.com/LSCO/${project}.git@${library_branch}#egg=${library_module}&subdirectory=${library_subdir}" \
            -t build/${project}
          ls -al build/${project}
          mkdir python && rsync -rv build/${project}/* python/
          zip -r9 ${layer_name}.zip python
          ls -al && pwd
          '''
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
        echo "Initializing python environment"
        ls -al && pwd
        cd ${module_name}
        ls -al && pwd
        cat .python-version
        PYENV_ROOT="$HOME/.pyenv"
        PATH="$PYENV_ROOT/bin:$PATH"
        eval "$(pyenv init -)"
        python3 -m venv --clear build && source build/bin/activate
        python3 --version
        which python && which python3 && which pip && which pip3
        pip3 install --upgrade pip
        
        echo "Creating artifacts from pip"
        mkdir artifacts
        pip3 download --no-deps --dest artifacts/ \
          "git+ssh://git@github.levi-site.com/LSCO/ds-lsa.git#egg=LeviDSTools&subdirectory=${module_name}/python3"
        export zipped=$(ls -1 artifacts/*.zip)
        
        echo "Publishing artifacts to s3"
        aws s3 cp $zipped s3://ds-lsa-${environment}/artifacts/

        echo "Autobots, roll out"
        docker build -t ${ecr_repo_name}:${tag} --rm -f Dockerfile .

        '''
      }
    } 

    stage('Confirm Deployment') {
      when {
        // Only say hello if a "greeting" is requested
        expression { params.require_approval == 'yes' }
      }
      steps {
        script {
          def userInput = input(id: 'confirm', message: 'Update AWS Lambda?', parameters: [ [$class: 'BooleanParameterDefinition', defaultValue: false, description: 'Update AWS Lambda', name: 'confirm'] ])
        }
      }
    }

    stage('Push') {
      steps {
        dir(build_dir) {
          sh "aws --region ${region} s3 cp ${layer_name}.zip s3://${bucket}/${layer_s3_key}/"
        }
      }
    }

    stage('Docker Login') {
      steps {
        sh '$(aws ecr get-login --no-include-email --region ${region})'
      }      
    }

    stage('Deploy') {
      steps {
        sh '''
          aws --region ${region} lambda publish-layer-version \
            --layer-name ${layer_name} \
            --content S3Bucket=${bucket},S3Key=${layer_s3_key}/${layer_name}.zip
          
          echo "Tagging remote image"
          docker tag ${ecr_repo_name}:${tag} ${ecr_endpoint}:${tag}
          
          echo "Pushing Docker image to ECR"
          docker push ${ecr_endpoint}:${tag}
        '''
      }
    }

    stage('Docker clean') {
      steps {
        sh '''
          echo "Removing local image"
          docker rmi -f ${ecr_endpoint}:${tag}
          docker rmi -f ${ecr_repo_name}:${tag}
        '''
      }
    }
  }

  post { 
    always { 
      cleanWs()
    }
  }
}
