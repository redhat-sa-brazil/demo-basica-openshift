# DEMO BÁSICA OPENSHIFT 3.11

SLIDES:
https://docs.google.com/presentation/d/1LzDT0TK7PJOsLNPipRNlQcjm0ecikLD6FKXF5lXinEs/edit#slide=id.g297305db33_0_0



## Parte 1

{nbsp} +

### Criar repositório *demo-openshift* no github:

* Acessar o seu GitHub particular:
  Novo repositorio::
    demo-openshift
       [x] Initialize this repository with README

* Criar um novo arquivo (index.php) e clicar em *commit*
* Código-fonte:

----
<?php
echo "<h1>Olá Mundo v1.0</h1> ";
echo "IP Interno: " . $_SERVER['SERVER_ADDR'];
echo "<br>";
?>
----

{nbsp} +

### Criar app no Openshift

	* Criar projeto "cliente"
	** Criar app "hello-world" PHP 7.0
		Name::   hello-world
		Git Repository:: https://github.com/guaxinim/demo-openshift.git

	* Logar pela console com o "Copy Login Command"
	* Mostrar node que ela "caiu"
	**	`oc project cliente`
	**	`oc get po -o wide`
	* Mostrar metricas, log, eventos, variaveis de ambiente, etc

{nbsp} +

### Criar um banco de dados
	* MySQL Ephemeral
	   Add to project:: MySQL Ephemeral
	* Criar banco no openshift
	   user admin, pass:: redhat@123
	   user root, pass:: redhat@123

	* Port forward para conectar ao MySQL
		** `oc project cliente`
		** `POD_MYSQL=$(oc get po | grep mysql | cut -f1 -d ' ')`
		** `oc port-forward $POD_MYSQL 3306:3306`

	* Criar base e popular dados
		** `mysql -u root -h 127.0.0.1 -p`

		**  `show databases;`
		**	`CREATE DATABASE clientedb;`
		**	`GRANT ALL PRIVILEGES ON clientedb.* TO 'admin'@'%' WITH GRANT OPTION;`
		**	`USE clientedb;`
		**	`CREATE TABLE cidade (id INT, nome VARCHAR(50));`
		**	`INSERT INTO cidade (id,nome) VALUES(1,"Brasilia");`
		**	`INSERT INTO cidade (id,nome) VALUES(2,"Sao Paulo");`
		**	`INSERT INTO cidade (id,nome) VALUES(3,"Rio de Janeiro");`
		**	`\q`

{nbsp} +


### Build e deploy automático através de Webhook
  * Cadastrar webhook
	** Settings -> Webhooks -> Add Webhook
		*** Copiar *GitHub Webhook URL* do Openshift

	** Colar a URL em "Payload"

	** Content type "application/json"

	** Disable SSL verification

	** Clicar em *Add webhook*


### Fazer rollout de nova versão da aplicação com acesso ao banco de dados

    ** Incluir Banco de Dados, versão 2.0 no index.php e fazer o commit

----
<?php
echo "<h1>Olá Mundo v2.0</h1> ";
echo "IP Interno: " . $_SERVER['SERVER_ADDR'];
echo "<br>";

$conn = new mysqli("mysql", "admin", "redhat@123", "clientedb");
if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

$result = $conn->query("SELECT nome FROM cidade");

if ($result->num_rows > 0) {
    while($row = $result->fetch_assoc()) {
        echo "<br>" . $row["nome"];
    }
} else {
    echo " - 0 results";
}
$conn->close();
?>
----


{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +


## Parte 2

{nbsp} +

### Escalar aplicação

	* Mostrar os nodes que ela caiu
	* Mostrar o balanceamento de carga

		oc get route

		while [ true ]; do curl http://hello-world-cliente.apps.example.com; sleep 1; echo; done

	* Escalar para 5
		** `oc get po -o wide`
	* Clicar no Pod para mostrar as métricas agregadas
	* Clicar em um Pod e Logging, mostrar *View Archive* em uma nova aba para abrir o Kibana
	* Tentar matar dois containers da aplicação
	* Voltar para 1 pod

{nbsp} +

### Checkagem da saúde (liveness e readiness) e debug do container
	* Escalar para 2 instancias
	* `while [ true ]; do curl http://hello-world-cliente.apps.example.com; sleep 1; echo; done`
    * Criar pagina liveness.php no GOGS

----
<?php
$filename = '/tmp/liveness';

if (file_exists($filename)) {
    header("HTTP/1.1 500 Internal Server Error");
} else {
    echo "Ok";
}
?>
----
	
* Criar a pagina readiness.php no GOGS

----
<?php
$filename = '/tmp/readiness';

if (file_exists($filename)) {
    header("HTTP/1.1 500 Internal Server Error");
} else {
    echo "Ok";
}
?>
----

	* git add and commit
	* Adicionar health check pela console web
	    ** Deployments:: Hello-world
	    ** Edit Health Check
		*** `/readiness.php` Delay 1 second
		*** `/liveness.php`. Delay 10 seconds
	* Fazer debug do container com readiness
		** Criar arquivo /tmp/readiness   (Tira do balanceamento)
		*** `touch /tmp/readiness`
		** Ir em eventos e ver o Probe Failed
		** Entrar no Pod com problema e clicar em 'Debug in Terminal'
		** Criar /tmp/liveness     (Container morreu, outro foi criado)
		*** `touch /tmp/liveness`

{nbsp} +

### Container em Stand-By (Idle)
	* Parar o `while [ true ]; do curl http://hello-world-cliente.apps.example.com/; sleep 1; echo; done`
	* `oc idle hello-world`
	* Abre o link da app no browser

{nbsp} +

### Limite de recursos
	* Escalar para 1
	* Application - Deployments - Hello World
	**    Colocar limit e request de memoria e cpu
	**	  cpu: 200           20% de cpu (o mesmo para request e limit)
	**	  memoria: 100m                  (o mesmo para request e limit)

	Fork Bomb (copiar do arquivo original):

	----
	while :; do _+=( $((++__)) ); done
	----
	
	** docker stats
	** Esperar ele matar o container
	** oc delete po <pod>

{nbsp} +

### Auto scaling
	* add auto scaler pela console
		 min:: 1
		 max:: 5
		 CPU request:: 20%
	* `ab -n 100000 -c 50 http://hello-world-cliente.apps.example.com/`
	* `oc delete hpa hello-world`

{nbsp} +

### Rolling update
	* Garantir que existam pelo menos 4 instancias
	* Forçar um novo deployment  (Applications - Deployments - Hello World)
	* Mostrar Rollback
		** Ir num deployment antigo e mostrar opção rollback
		** Marcar todas as opções

{nbsp} +

### A/B Testing
	* Desagrupar o banco
	* Remover webhook
	* Criar branch no git
		** `cd /tmp`
		** `git clone https://github.com/guaxinim/demo-openshift.git`
		** `cd demo-openshift`
		** `git branch v3.0`
		** `git checkout v3.0`
		** Alterar (index.php) para versão 3.0
		** `git commit -am "v3.0"`
		** `git push origin v3.0`

	* Add to project
		** PHP
		** `hello-world-v3`
		** `https://github.com/guaxinim/demo-openshift.git`
		** Advanced
		** branch `v3.0`

	* `while [ true ]; do curl http://hello-world-cliente.apps.example.com; sleep 1; echo; done`
	* Abrir rota hello-world antiga
	**   Edit
	    ***   Split traffic accross multiple services
	    *** Alterar a porcentagem de cada um

{nbsp} +

### Blue Green Deployment
	* Remove ab testing
		** Application -> Routes
		*** hello-world
		  **** Edit
		  ****  Desmarcar (Split traffic across multiple services)
	* Application -> Routes
	**   hello-world
	***       Edit
	****           Escolher - Service: `hello-world-v3`

{nbsp} +

### Jenkins

    * Deletar projeto cliente. +
      `oc delete project cliente`

    * Entrar no projeto CI-CD
	* Subir jenkins  (ephemeral)
	* Criar Jenkinsfile no repo

----
node('maven') {
	def app = 'hello-world'
	def cliente = 'cliente'
	def gitUrl = 'https://github.com/guaxinim/demo-openshift.git'

    stage 'Build image and deploy to dev'
    echo 'Building docker image'
    buildApp(app + '-dev', gitUrl, app)

    stage 'Deploy to QA'
    echo 'Deploying to QA'
    deploy(app + '-dev', app + '-qa', app)

    stage 'Wait for approval'
    input 'Aprove to production?'

    stage 'Deploy to Prod'
    deploy(app + '-dev', app + '-prd', app)
}

def buildApp(String project, String gitUrl, String app){
    projectSet(project)

    sh "oc new-build php:7.0~${gitUrl} --name=${app} -n ${project}"
    sh "oc logs -f bc/${app} -n ${project}"
    sh "oc new-app ${app} -n ${project}"
    sh "oc expose service ${app} -n ${project} || echo 'Service already exposed'"
    sh "sleep 10"
}

def projectSet(String project) {
	sh "oc login -u admin -p redhat@123 https://master.example.com:8443"
	sh "oc login --insecure-skip-tls-verify=true https://master.rhpds311.openshift.opentlc.com:443 --token=Miu0-D9uNqfVOmyBlLLB8lYWhrL18nFXVxFi7yw5z_c"
    sh "oc new-project ${project} || echo 'Project exists'"
    sh "oc project ${project}"
}

def deploy(String origProject, String project, String app){
    projectSet(project)
    sh "oc policy add-role-to-user system:image-puller system:serviceaccount:${project}:default -n ${origProject}"
    sh "oc tag ${origProject}/${app}:latest ${project}/${app}:latest"
    appDeploy(app, project)
}

def appDeploy(String app, String project){
    sh "oc new-app ${app} -l app=${app} -n ${project} || echo 'Aplication already Exists'"
    sh "oc expose service ${app} -n ${project} || echo 'Service already exposed'"
}
----

* Mudar para projeto Cliente
* Criar buildconfig para o Pipeline  (Add to project - From yaml/json)

----
kind: "BuildConfig"
apiVersion: "v1"
metadata:
  name: "php-pipeline"
  annotations:
    pipeline.alpha.openshift.io/uses: '[{"name": "hello-world", "namespace": "", "kind": "DeploymentConfig"}]'
spec:
  source:
    type: "Git"
    git:
      uri: "https://github.com/guaxinim/demo-openshift.git"
  strategy:
    type: "JenkinsPipeline"
    jenkinsPipelineStrategy:
      jenkinsfilePath: ""
----

* Mostrar pipeline no Openshift e no Jenkins
* Dar um start pipeline no Openshift


{nbsp} +


### Desenvolvimento direto no container
	* Clonar repo na maquina local
	    ** `cd /tmp`
		** `git clone http://gogs-ci-cd.apps.example.com/gogsadmin/cliente.git`
		** `oc login https://master.example.com:8443 -u admin`

	* Conectar diretamente ao pod e alterar o código
		** `oc project cliente`
		** `POD_PHP=$(oc get po --show-all=false | grep hello-world | cut -f1 -d ' ')`
		** `oc rsync /tmp/cliente/ $POD_PHP:/opt/app-root/src -w --no-perms=true`
	* Alterar código-fonte e ver a mudança direto na aplicação

----
<?php
echo "<h1>Olá Mundo v3.0</h1> ";
echo $_SERVER['SERVER_ADDR'];

$conn = new mysqli("mysql", "admin", "redhat@123", "clientedb");
if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

$result = $conn->query("SELECT nome FROM cidade");

if ($result->num_rows > 0) {
    while($row = $result->fetch_assoc()) {
        echo "<br>" . $row["nome"];
    }
} else {
    echo " - 0 results";
}
$conn->close();
?>
----

	* Parar o rsync
		** `cd /tmp/cliente`
		** `git add *`
		** `git commit -m "add db"`
		** `git push origin master`

	* Builds -> Builds -> Start Build

{nbsp} +


### Mostrar Cockpit

	* https://master.example.com:9090




{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +
{nbsp} +

'''




{nbsp} +

### Job
	* Inserir o Job no projeto cliente
----
apiVersion: batch/v1
kind: Job
metadata:
  name: job
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
	      containers:
	      - name: hello
	        image: busybox
	        args:
          - /bin/sh
          - -c \
          - date; echo Hello from the Kubernetes cluster
	      restartPolicy: OnFailure
----
----
apiVersion: batch/v2aplha1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            parent: "cronjobhello"
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["/bin/sh", "-c", "date;", "echo Hello from Openshift Container Platform"]
	        restartPolicy: OnFailure
----

Gogs

	mkdir /exports/gogs
	chmod 777 /exports/gogs
	chown nfsnobody: /exports/gogs

	vi /etc/exports.d/openshift-ansible.exports
		/exports/gogs *(rw,no_root_squash)

	Abrir cockpit:	https://master.example.com:9090
	gogs
	1Gi
	RWX
	Retain
	master.example.com
	/exports/gogs

	oc new-project ci-cd --name='CI CD Tools'
	oc adm policy add-scc-to-user anyuid -z default -n ci-cd
	Openshift console:
		Storage
			Create new PVC
			1Gi - RWO

	gogs deployment - add storage:


Roadmap Openshift (Interno):

	https://docs.google.com/presentation/d/1_4VKf_RsvX1_-qiKv81KkWlbdIakOrrSmjjSG8PrihI/edit#slide=id.g23878e0145_0_0




	raw:   while :; do _+=( $((++__)) ); done
	
	* Logar no node que ele estiver via ssh
	** `docker ps`
	** `docker stats`
	** Esperar ele matar o container (sem resposta http)
	** `oc delete po <pod>`


*Outros*

9) Rollback
				-- Fazer rollback pela web console
					-- marcar todas as opções

10) Security com ImageStream
	-- Criar imagem docker do php
	-- Fazer push para o registry
	-- Criar image stream
	-- Criar o build config com base na imagem anterior
	-- Executar o build config


11) Evacuate
	-- oadm manage-node ocp-node01.example.com --evacuate --dry-run




