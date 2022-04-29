# Specification of the microservice working scheme

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
docker run --network=kv126 -d --name json-filter -e HOST="0.0.0.0" -e PORT="5000" -e QUEUE_SLACK="slack" -e QUEUE_RESTAPI="restapi" -e TOKEN=<tocken for slack> -e CHANNEL="rabbit-to-slack" -e RMQ_HOST=rabbit -e RMQ_PORT=5672 -e RMQ_LOGIN=mquser -e RMQ_PASS=mqpass -p 5000:5000 -v /opt/json-filter:/app python:3.9-slim sleep infinity
docker exec json-filter pip install pika flask python-dotenv
docker exec -d json-filter bash -c "cd /app && python ./jsonfilter.py"

### RabbitMQ-to-DB		https://github.com/Kv-126-DevOps/rabbit-to-bd.git

### RabbitMQ-to-slack	https://github.com/Kv-126-DevOps/rabbit_to_slack.git

### Rest API			https://github.com/Kv-126-DevOps/rest-api.git
### Frontend			https://github.com/Kv-126-DevOps/frontend.git
