#!/usr/bin/env groovy
@Library(['piper-lib', 'piper-lib-os','piper-refapps']) _

node{
	dockerExecuteOnKubernetes(script: this, dockerEnvVars: ['pusername':pusername, 'puserpwd':puserpwd], dockerImage: 'docker.wdf.sap.corp:51010/sfext:v3' ) {

	try {
		stage ('Build') { 
			deleteDir()
      		checkout scm
			
	 		sh '''
			    npm config set unsafe-perm true
			    npm rm -g @sap/cds
			    npm i -g @sap/cds-dk
			    cd tests/mocks
			    cds build --production
			    cd ../..
			    mv ./build/manifest1.yml tests/mocks/gen/srv/manifest.yml
				mv ./build/xs-security.json xs-security.json
			'''
			packageJson = readJSON file: 'package.json'
			packageJson.cds.requires.API_BUSINESS_PARTNER["[production]"].credentials.destination = "bupa-mock"
			writeJSON file: 'package.json', json: packageJson
			sh "cat package.json"
			sh "mbt build -p=cf"
			
		 
	  	}

		stage('Test: Unit'){
			sh "npm run-script test:unit"
		}

		stage('Test: Integration') {
			sh "npm run-script test"
		}

	  	stage('Deploy Mock'){
			setupCommonPipelineEnvironment script:this
			cloudFoundryDeploy script:this, deployTool:'cf_native', cloudFoundry: [manifest: 'tests/mocks/gen/srv/manifest.yml']
			cloudFoundryDeploy script:this, deployTool:'mtaDeployPlugin'  	
        }

		stage('Test: REST-API'){
			catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
				withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId:'pusercf', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
					sh "cf login -a ${commonPipelineEnvironment.configuration.steps.cloudFoundryDeploy.cloudFoundry.apiEndpoint} -u $USERNAME -p $PASSWORD -o ${commonPipelineEnvironment.configuration.steps.cloudFoundryDeploy.cloudFoundry.org} -s ${commonPipelineEnvironment.configuration.steps.cloudFoundryDeploy.cloudFoundry.space}"
				}
				sh '''
					appId=`cf app BusinessPartnerValidation-srv --guid`
					`cf curl /v2/apps/$appId/env > tests/rest-api/testscripts/util/appEnv.json`
					npm install --only=dev
					npm run-script test:rest-api
				'''
			}
	    }
	  
	   	stage('Redeploy'){
		   	sh "rm -rf *"
      		checkout scm
		   	sh '''
			    mv ./build/xs-security.json xs-security.json
			    mbt build -p=cf
			'''
			cloudFoundryDeploy script:this, deployTool:'mtaDeployPlugin'  
	    } 

 	   	stage('UI Test'){
		   
			build job: 'customlogicS4_demoscript'
		
		}
	   	stage('Undeploy'){
			cloudFoundryDeleteApp (script: this, btpCredentialsId: 'pusercf', orgName: 'SA0176014160_refapps-cicd', spaceName: 'CICD_s4ext', btpRegion: 'eu10', appName: 'BusinessPartnerValidation-srv-mocks')
			cloudFoundryUndeploy (script: this, btpCredentialsId: 'pusercf', orgName: 'SA0176014160_refapps-cicd', spaceName: 'CICD_s4ext', btpRegion: 'eu10', appName: 'BusinessPartnerValidation')
			}
	}
	catch(e){
		echo 'This will run only if failed'
		currentBuild.result = "FAILURE"
	}
	finally {
		 emailext body: '$DEFAULT_CONTENT', subject: '$DEFAULT_SUBJECT', to: 'DL_5731D8E45F99B75FC100004A@global.corp.sap,DL_58CB9B1A5F99B78BCC00001A@global.corp.sap'
	}
}
}
