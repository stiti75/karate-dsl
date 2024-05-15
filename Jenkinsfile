pipeline {
	agent {
        label 'jdk17'
    }

	environment {
		MAVEN_ENV = 'Maven 3.5.2'
		MAVEN_SETTINGS = '2db35d6c-b2d3-4c1d-8755-407661631a46'
        credentials_artifactory = credentials('jenkins-user-artifactory-laforge')
		selenium_auth = credentials('jenkins-user-selenium-laforge')
		browserstack_auth = credentials('jenkins-user-browserstack-laforge')
		XRAY_TOKEN = credentials('xray-token-api')
	}

	parameters {
                           string(name: 'ISSUE_XRAY_KEY', defaultValue: 'TAO-1664')
                           choice(name: 'ENV', choices: ['staging', 'master-staging', 'dev'], description: 'Environnement technique')
                           string(name: 'JWT_FULL', defaultValue: '')


                   }

	options {
		gitLabConnection("gitlab")
	}

	stages {

		/**
		* ce stage permet de télécharger les fichier de test (features) et les mettre dans le dossier src\test\resources\features
		* une fois les fichier récupérés, il adapte les features pour que ça soit en anglais
		* @param: XRAY_TOKEN
		* @param: ISSUE_XRAY_KEY
		*/
		stage('Get Features From Xray'){
			steps {
				script{
					//withVault(vaultSecrets: [[path: 'team-tao-kv2/jenkins/jira-ppd', secretValues: [[envVar: 'mytoken', vaultKey: 'xray_token']]]]) {env.XRAY_TOKEN = "${mytoken}"}
					//
					sh "curl -H 'Content-Type: application/json' -H 'Authorization: Bearer ${XRAY_TOKEN}' -X GET 'https://jira.web.bpifrance.fr/rest/raven/1.0/export/test?keys=${ISSUE_XRAY_KEY}&fz=true' --output features.zip"
					sh "ls -la"
					sh "mkdir -p -v src/test/resources/features"
					unzip zipFile: 'features.zip', dir: 'temp'
					def features = findFiles(glob: 'temp/*.feature')
					for (int i = 0; i < features.size(); i++) {
						// je formate chaque feature afin que la langue gherkin soit de l'anglais au lieux du français (karate n'est pas compatible avce les autres langues)
						sh "cat temp/${features[i].name} | sed s/#language.*//g | sed s/Fonc.*\\:/Feature\\:/g | sed s/Contexte\\:/Background\\:/g | sed s/Sc.*\\:/Scenario\\:/g | sed s/Plan.*\\:/'Scenario Outline:'/g | sed s/Exemples\\:/Examples\\:/g | sed s/Soit/Given/g | sed s/Quand/When/g | sed s/Alors/Then/g | sed s/Et/And/g> src/test/resources/features/${features[i].name}"
						// j'affiche le contenu de chaque feature
						sh "cat src/test/resources/features/${features[i].name}"
					}
				}
			}
		}

		stage('Download Attachments From Test Cases'){
			/**
			* ce stage permet de télécharger des pièces jointes au cas de tests, elle seront stockées dans src\test\resources\data
			* @param: XRAY_TOKEN
			* @param: ISSUE_XRAY_KEY
			* @param: MATRICULE
			* @param: MDP
			*/
			steps {
				script{
					sh "mkdir -p -v src/test/resources/data"
                    //Renvoi le type de ticket donné en paramètre (Test|Test Set|Test Plan|Test Execution) et pour supprimer le champ test plan si nécessaire
                    env.ISSUE_XRAY_TYPE = sh(script:"curl -s -H 'accept: application/json' -H 'Authorization: Bearer ${XRAY_TOKEN}' -X GET https://jira.web.bpifrance.fr/rest/api/2/issue/${ISSUE_XRAY_KEY} |  jq -r .fields.issuetype.name", returnStdout:true).trim();

			        def tests
					if(env.ISSUE_XRAY_TYPE == 'Test Plan'){
						//je récupère la liste des clé des cas de test rattachés à un test plan
                        tests =  sh(script:"curl -H 'Authorization: Bearer ${XRAY_TOKEN}' GET 'https://jira.web.bpifrance.fr/rest/api/2/issue/${ISSUE_XRAY_KEY}' | jq -r .fields.customfield_11827" , returnStdout:true).trim().replaceAll('\\[','').replaceAll('\\]','').replaceAll('"','').split(',');
			        }else if(env.ISSUE_XRAY_TYPE == 'Test Execution'){
						//je récupère la liste des clé des cas de test rattachés à un test execution
						tests =  sh(script:"curl -H 'Authorization: Bearer ${XRAY_TOKEN}' GET 'https://jira.web.bpifrance.fr/rest/api/2/issue/${ISSUE_XRAY_KEY}' | jq -r .fields.customfield_11816[].b" , returnStdout:true).trim().replaceAll('\\[','').replaceAll('\\]','').replaceAll('"','').split('\n');
					}

					//je parcours chaque test et j'extracte les pièces jointes si elle existent
					for(String test: tests){
						def testKey = test.trim()
						//je récupère la liste des url des pièces jointes
						def urls = sh(script:"curl -H 'Authorization: Bearer ${XRAY_TOKEN}' GET 'https://jira.web.bpifrance.fr/rest/api/2/issue/${testKey}' | jq -r .fields.attachment[].content" , returnStdout:true).trim();
						//je récupère la liste des noms des pièces jointe
						def names = sh(script:"curl -H 'Authorization: Bearer ${XRAY_TOKEN}' GET 'https://jira.web.bpifrance.fr/rest/api/2/issue/${testKey}' | jq -r .fields.attachment[].filename" , returnStdout:true).trim();
						if ( ! urls.toString().isEmpty()){
							for(int i = 0; i < urls.toString().split('\n').length; i++){
								def url = urls.toString().split('\n')[i].trim()
								def name = names.toString().split('\n')[i].trim()
								//je télécharge la pièce jointe
								sh(script:"curl -H 'Content-Type: application/json' -u 'MA0014:xBh94Qm4T' GET '${url}' > 'src/test/resources/data/${name}'", returnStdout:true).trim()
								//j'affiche le contenu de la pièce jointe
								sh "cat 'src/test/resources/data/${name}'"
							}
						}
					}
				}
			}
    	}

		stage('Run Tests') {
			/**
			*ce stage permet d'exécuter les test java avec maven peu importe le type de test (ihm, api, data)
			*@param: SELENIUM_BROWSER (jenkins: type du browser)
			*@param: SELENIUM_FRONT_URL (jenkins ou xray : url du test IHM)
			*@param: SELENIUM_HUB_URL (jenkins: url du hub selenium ou browserstack)
			*@param: URL_API
			*@param: MVN_PARAMS
			*/
			steps {
				catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
					configFileProvider([configFile(fileId: 'global-main-artifactory-la-forge', variable: 'MAVEN_SETTINGS_XML')]) {
						script {
							if (params.MVN_PARAMS != null){
								sh 'mvn -s $MAVEN_SETTINGS_XML clean verify -Dfile.encoding=utf-8 -U -DSELENIUM_BROWSER=' + params.SELENIUM_BROWSER + ' -DSELENIUM_FRONT_URL=' + params.SELENIUM_FRONT_URL + ' -DSELENIUM_HUB_URL=https://${selenium_auth_USR}:${selenium_auth_PSW}@' + params.SELENIUM_HUB_URL  + ' -DURL_API=' + params.URL_API + ' ' + params.MVN_PARAMS
							}else{
								sh 'mvn -s $MAVEN_SETTINGS_XML clean verify -Dfile.encoding=utf-8 -U -DSELENIUM_BROWSER=' + params.SELENIUM_BROWSER + ' -DSELENIUM_FRONT_URL=' + params.SELENIUM_FRONT_URL + ' -DSELENIUM_HUB_URL=https://${selenium_auth_USR}:${selenium_auth_PSW}@' + params.SELENIUM_HUB_URL  + ' -DURL_API=' + params.URL_API
							}
						}
					}
				}
			}
		}

        stage('Send Résults to Xray') {
			/**
			* ce stage gère la remontée des résultat vers Xray
			* la première étape consiste à créer une nouvelle test exécution
			* ensuite on va générer un fichier json contenant les informations à envoyer à cet nouvelle exécution
			* @param: XRAY_TOKEN
			* @param: ISSUE_XRAY_KEY (jenkins ou xray)
			* @param: SELENIUM_FRONT_URL (jenkins ou xray : url du test IHM)
			* @param: URL_API (jenkins ou xray: url du test api)
			* @param: ISSUE_XRAY_TYPE (récupéré du stage précédent)
			*/
            steps {
                script{
					//zipper les fichier de résultat de test .json dans un fichier zip avec le nom results.zip
                    try{
                        sh 'zip results.zip target/karate-reports/*.json'
                    }catch(Exception e){
                        sh 'zip results.zip target/surefire-reports/*.json'
                    }
					//envoyer le fichier zip vers xray ==> créer une nouvelle exécution
					env.NEW_TEST_EXEC_KEY = sh(script:"curl -s -H 'accept: application/json' -H 'Content-Type: multipart/form-data' -H 'Authorization: Bearer ${XRAY_TOKEN}' -F 'file=@results.zip' POST https://jira.web.bpifrance.fr/rest/raven/1.0/import/execution/bundle |  jq -r .testExecIssue.key", returnStdout:true).trim();
					echo "${NEW_TEST_EXEC_KEY}"
					//obtenir le nom du ticket jira en question
					env.ISSUE_NAME = sh(script:"curl -H 'Authorization: Bearer ${XRAY_TOKEN}' GET 'https://jira.web.bpifrance.fr/rest/api/2/issue/${ISSUE_XRAY_KEY}' | jq -r .fields.summary" , returnStdout:true).trim();


					//si le champ url de xray n'est pas vide alors j'envoi sa valeur dans le rapport
					echo "----------------> vérification du param SELENIUM_FRONT_URL"
					if( params.SELENIUM_FRONT_URL != null ){
						if(! params.SELENIUM_FRONT_URL.toString().trim().isEmpty()){
							echo "----------------> URL FRONT n'est pas vide"
							sh '''
								sed -r 's@<url_ihm>@'"${SELENIUM_FRONT_URL}"'@g' xray-infos.json > temp.json; mv -f temp.json xray-infos.json
								cat xray-infos.json
							'''
						}
					}else{
						//si le champ url est vide, alors je supprime l'info du fichier infos-xray.json
						echo "----------------> URL FRONT vide, je supprime le champ du fichier json"
						sh '''
							sed "/customfield_14602/d" xray-infos.json > temp.json; mv -f temp.json xray-infos.json
						'''
					}
					//si le champ url api de xray n'est pas vide alors j'envoi sa valeur dans le rapport
					echo "----------------> vérification du param URL_API"
					echo params.URL_API
					if(params.URL_API != null ){
						echo "----------------> URL_API n'est pas null"
						if(params.URL_API.toString().contains('http')){
							echo "----------------> URL API n'est pas vide "
							sh '''
								sed -r 's@<url_api>@'"${URL_API}"'@g' xray-infos.json > temp.json; mv -f temp.json xray-infos.json
							'''
						}else{
							// si le champ url api est vide alors je supprime l'info du fichier xray-infos.json
							echo "----------------> URL API vide, je supprime le champ du fichier json"
							sh '''
								sed "/customfield_14603/d" xray-infos.json > temp.json; mv -f temp.json xray-infos.json
							'''
						}
					}else{
						// si le champ url api est vide alors je supprime l'info du fichier xray-infos.json
						echo "----------------> URL API vide, je supprime le champ du fichier json"
						sh '''
							sed "/customfield_14603/d" xray-infos.json > temp.json; mv -f temp.json xray-infos.json
						'''
					}

					//je construis mon fichier json des infos à envoyé à xray en fonction du type du ticket
					if (env.ISSUE_XRAY_TYPE == 'Test Plan'){
						//construire le nom de la nouvelle test exécution  créée dans Xray
						echo "----------------> Reporting pour TEST PLAN"
						sh '''
							cat xray-infos.json |sed -r "s/<type>/${ISSUE_XRAY_TYPE}/g" | sed -r "s/<key>/[${ISSUE_XRAY_KEY}]/g" | sed -r "s/<build>/ ${BUILD_NUMBER} # ${ISSUE_NAME}/g" | sed -r "s/<job>/${JOB_NAME}/g" | sed -r "s/<testPlanKey>/${ISSUE_XRAY_KEY}/g" | sed -r 's@<buil_url>@'"${BUILD_URL}"'@g' > temp.json
							mv temp.json xray-infos.json
							cat xray-infos.json
						'''
						// envoie du rapport
						sh "curl -H 'Content-Type: application/json' -H 'Authorization: Bearer ${XRAY_TOKEN}' -X PUT  --data '@xray-infos.json' https://jira.web.bpifrance.fr/rest/api/2/issue/${NEW_TEST_EXEC_KEY}"
						//gérer le cas d'une test exécution
					}else if(env.ISSUE_XRAY_TYPE == 'Test Execution'){
						echo "----------------> Reporting pour TEST EXECUTION"
						if(env.ISSUE_NAME.contains('#')){
							echo "----------------> Le nom du ticket est généré par l'automate"
							def count = env.ISSUE_NAME.count("#")
							if (count > 1){
								echo "----------------> le nom contient 2 #"
								env.NEW_ISSUE_NAME = env.ISSUE_NAME.split('#')[-1]
								env.ISSUE_XRAY_TYPE = env.ISSUE_NAME.split('#')[0]
								sh '''
									cat xray-infos.json | sed -r "s/<type>/${ISSUE_XRAY_TYPE}/g" | sed -r 's@ <key> @'""'@g' | sed -r "s/<build>/${BUILD_NUMBER} # ${NEW_ISSUE_NAME}/g" | sed -r "s/<job>/${JOB_NAME}/g" | sed -r 's@<buil_url>@'"${BUILD_URL}"'@g' > temp.json
									mv temp.json xray-infos.json
								'''
							}else{
								echo "----------------> Il s'agit de la mise à jour d'une test exécution existante créée par un humain"
								sh '''
									cat xray-infos.json | sed -r "s/<type>/${ISSUE_XRAY_TYPE}/g" | sed -r "s/<key>/[${ISSUE_XRAY_KEY}]/g" | sed -r "s/<build>/${BUILD_NUMBER}/g" | sed -r "s/<job>/${JOB_NAME}/g" | sed -r 's@<buil_url>@'"${BUILD_URL}"'@g' > temp.json
									mv temp.json xray-infos.json
								'''
							}

						}else{
							echo "----------------> Le nom du ticket est généré par l'utilisateur Jira"
							sh '''
								cat xray-infos.json | sed -r "s/<type>/${ISSUE_XRAY_TYPE}/g" | sed -r "s/<key>/[${ISSUE_XRAY_KEY}]/g" | sed -r "s/<build>/${BUILD_NUMBER}/g" | sed -r "s/<job>/${JOB_NAME}/g" | sed -r 's@<buil_url>@'"${BUILD_URL}"'@g' > temp.json
								mv temp.json xray-infos.json
								'''
						}

						echo "----------------> je supprime le champ Test Plan du fichier JSON"
						sh '''
							sed "/customfield_11828/d" xray-infos.json > temp.json; mv -f temp.json xray-infos.json
							cat xray-infos.json
						'''
						sh "curl -H 'Content-Type: application/json' -H 'Authorization: Bearer ${XRAY_TOKEN}' -X PUT  --data '@xray-infos.json' https://jira.web.bpifrance.fr/rest/api/2/issue/${ISSUE_XRAY_KEY}"
					}
				}
			}
        }
	}

	post ('Update the status of the Field Status Pipeline in Xray'){
		/**
		* ce stage permet de mettre à jour la valeur du champ ETAT DU PIPELINE
		*/

        aborted {
			script{
				sh "curl -X PUT 'https://jira.web.bpifrance.fr/rest/api/2/issue/${ISSUE_XRAY_KEY}' -H 'Authorization: Bearer ${XRAY_TOKEN}' -H 'Content-Type: application/json' --data-raw '{\"fields\": {\"customfield_14400\": { \"value\": \"Libre\"}}}'"
			}
        }

        always {
			script{
				sh "curl -X PUT 'https://jira.web.bpifrance.fr/rest/api/2/issue/${ISSUE_XRAY_KEY}' -H 'Authorization: Bearer ${XRAY_TOKEN}' -H 'Content-Type: application/json' --data-raw '{\"fields\": {\"customfield_14400\": { \"value\": \"Libre\"}}}'"
			}
            dir('target') {
                archiveArtifacts '**'
			}
		}
	}
}