# Specification of the microservice working scheme

### Create infrastructure
	docker network create -d bridge kv126
	docker run --network=kv126 -d --name postgres -e POSTGRES_USER=dbuser -e POSTGRES_PASSWORD=dbpass postgres:14
	docker run --network=kv126 -d --name rabbit -e RABBITMQ_DEFAULT_USER=mquser -e RABBITMQ_DEFAULT_PASS=mqpass -p 15672:15672 rabbitmq:3.9-management

### Github
	
	Integration with https://github.com/Kv-126-DevOps/None.git
	Configure webhook on the https://github.com/Kv-126-DevOps/None.git , specify:
	 - Payload URL -  is the URL of the server that will receive the webhook 
	   External IP of the docker with json filter ( I have done it without using ssl, specify http http://<my external IP>:5000 )
	 - Content type
	   application/json
	 - Which events would you like to trigger this webhook? 
	   Choose "Send me everything"
	
	Example of webhook creation (from Kv-126-DevOps/json-filter)
	 https://www.cidevops.com/2019/02/how-to-create-webhooks-in-github-and.html

	
### Json-filter			https://github.com/Kv-126-DevOps/json-filter.git
	
	Filter listens to port 5000. When added new records to Github repo , webhook sends "everything" to the json-filter
	
	Fields that will be selected (jsonfilter.py) :
	if action == 'opened':
        result_dict['action'] = action
        result_dict['issue'] = parseOpened(data, base_issue, base_user)            # selects required fields
    elif action == 'assigned':
        result_dict['action'] = action
        assignee = base_issue['assignee']
        result_dict['issue'] = parseOther(base_issue, assignee)                     # selects required fields
    else:
        result_dict['action'] = action
        result_dict['issue'] = parseOther(base_issue, base_user)
	
#### Think about what keys needed when docker container is started. For example in the repo json-filter/.env specified:
		# Flask webhook listener
		HOST = "0.0.0.0"
		PORT = "5000"
	
		# RabbitMQ
		RMQ_HOST = "rabbit"
		RMQ_PORT = "5672"
		RMQ_LOGIN = "mquser"
		RMQ_PASS = "mqpass"
		QUEUE_SLACK = "slack"
		QUEUE_RESTAPI = "restapi"
	
		# Slack
		TOKEN = Get it from https://api.slack.com/apps/A03DGUF01FE -> App-Level Tokens -> rabbit-to-slack
		CHANNEL = rabbit-to-slack
	
#### Working version:
	#Run json-filter

	git clone --branch develop https://github.com/Kv-126-DevOps/json-filter.git /opt/json-filter
	docker run --network=kv126 -d --name json-filter -e HOST="0.0.0.0" -e PORT="5000" -e QUEUE_SLACK="slack" -e QUEUE_RESTAPI="restapi" -e TOKEN="<Get it from https://api.slack.com/apps/A03DGUF01FE -> App-Level Tokens -> rabbit-to-slack>" -e CHANNEL="rabbit-to-slack" -e RMQ_HOST=rabbit -e RMQ_PORT=5672 -e RMQ_LOGIN=mquser -e RMQ_PASS=mqpass -p 5000:5000 -v /opt/json-filter:/app python:3.9-slim sleep infinity
	docker exec json-filter pip install pika flask python-dotenv
	docker exec -d json-filter bash -c "cd /app && python ./jsonfilter.py"

### RabbitMQ-to-DB		https://github.com/Kv-126-DevOps/rabbit-to-bd.git

    git clone --branch 1-rabbit-to-bd-code-refactoring https://github.com/Kv-126-DevOps/rabbit-to-bd.git /opt/rabbit-to-bd
    docker run --network=kv126 -e POSTGRES_HOST=postgres -e POSTGRES_PORT=5432 -e POSTGRES_USER=dbuser -e POSTGRES_PW=dbpass -e POSTGRES_DB=postgres -e RABBIT_HOST=rabbit -e RABBIT_PORT=5672 -e RABBIT_USER=mquser -e RABBIT_PW=mqpass -e RABBIT_QUEUE=restapi -d --name rabbit-to-bd -v /opt/rabbit-to-bd:/app python:3.9-slim sleep infinity
    docker exec rabbit-to-bd pip install -r /app/requirements.txt
    docker exec -d rabbit-to-bd bash -c "cd /app && python ./app.py"

#### Check of the working capacity 
	When RabbitMQ-to-DB, json-filter started (with "Create infrastructure" param), records of issues created in the github should be added to postgre DB. When connect to DB container , check: 
	psql -d postgres -U dbuser -W
	\dt+
	and for example TABLE "Issues";
	
### RabbitMQ-to-slack	https://github.com/Kv-126-DevOps/rabbit_to_slack.git

	In the slack application configure "Incoming Webhooks to slack" create Webhook URL that will be as SLACK_URL variable  [link]	(https://medium.com/@sharan.aadarsh/sending-notification-to-slack-using-python-8b71d4f622f3)
	Channel for slack is specified during "Incoming Webhooks to slack" configuration.

#### Run rabbit-to-slack
    git clone --branch 1-rabbit-to-slack-code-refactoring https://github.com/Kv-126-DevOps/rabbit_to_slack.git /opt/rabbit-to-slack
    docker run --network=kv126 -e RABBIT_HOST=rabbit -e RABBIT_PORT=5672 -e RABBIT_USER=mquser -e RABBIT_PW=mqpass -e RABBIT_QUEUE=slack -e SLACK_URL="<Get from https://devopskv-126.slack.com/services/3459809187923?updated=1 -> Webhook URL>" -d --name rabbit-to-slack -v /opt/rabbit-to-slack:/app python:3.9-slim sleep infinity
    docker exec rabbit-to-slack  pip install -r /app/requirements.txt
    docker exec -d rabbit-to-slack bash -c "cd /app && python ./app.py"

    When  json-filter and rabbit-to-slack are configured , messages for new issues in the connected Github repo will be appeared in the slack channel "rabbit-to-slack".
    When specify SLACK_URL and TOKEN in the GitHub, it is unclear why locked and should be recreated.
    
### Rest API			https://github.com/Kv-126-DevOps/rest-api.git
	git clone --branch 14-rest-api-code-refactoring https://github.com/Kv-126-DevOps/rest-api.git /opt/rest-api 
	docker run --network=kv126 -d --name rest-api -e POSTGRES_HOST=postgres -e POSTGRES_PORT=5432 -e POSTGRES_USER=dbuser -e POSTGRES_PASS=dbpass -e POSTGRES_DB=postgres -v /opt/rest-api:/app -p 8080:5000 python:3.9-slim sleep infinity 
	docker exec rest-api pip install -r /app/requirements.txt docker exec -d rest-api bash -c "cd /app && flask run --host=0.0.0.0"
	 
### Frontend			https://github.com/Kv-126-DevOps/frontend.git
	git clone --branch 1_frontend_code_refactoring https://github.com/Kv-126-DevOps/frontend.git /opt/frontend 
	docker run --network=kv126 -d --name frontend -e RESTAPI_HOST=rest-api -e RESTAPI_PORT=5000 -v /opt/frontend:/app -p 80:5000 python:3.9-slim sleep infinity 
	docker exec frontend pip install -r /app/requirements.txt docker exec -d frontend bash -c "cd /app && flask run --host=0.0.0.0"
