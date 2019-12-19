#! /usr/bin/make -f

.DEFAULT_GOAL := help

INSTANA_AGENT_ZONE := INSTANA_AGENT_ZONE
INSTANA_AGENT_KEY := INSTANA_AGENT_KEY
INSTANA_AGENT_ENDPOINT := INSTANA_AGENT_ENDPOINT
INSTANA_AGENT_ENDPOINT_PORT := INSTANA_AGENT_ENDPOINT_PORT

#
# Part 0: Setup
#

install-operator:
	git clone https://github.com/instana/instana-agent-operator
	kubectl apply -f instana-agent-operator/deploy/instana-agent-operator.yaml
	sed -i '' 's/(optional) name of the zone of the host/$(INSTANA_AGENT_ZONE)/g' instana-agent-operator/deploy/instana-agent.customresource.yaml
	sed -i '' 's/put your Instana agent key here/$(INSTANA_AGENT_KEY)/g' instana-agent-operator/deploy/instana-agent.customresource.yaml
	sed -i '' 's/the monitoring ingress endpoint port, wrapped in quotes/$(INSTANA_AGENT_ENDPOINT_PORT)/g' instana-agent-operator/deploy/instana-agent.customresource.yaml
	sed -i '' 's/the monitoring ingress endpoint/$(INSTANA_AGENT_ENDPOINT)/g' instana-agent-operator/deploy/instana-agent.customresource.yaml
	kubectl apply -f instana-agent-operator/deploy/instana-agent.customresource.yaml

delete-operator:
	kubectl delete namespace instana-agent
	kubectl delete CustomResourceDefinition agents.instana.io
	rm -rf instana-agent-operator

# yes it's to complicated
logs-operator: OPERATOR_POD = $(shell kubectl get pods -n instana-agent -l app=instana-agent-operator -o json | jq .items[0].metadata.name)
logs-operator:
	kubectl logs $(OPERATOR_POD) -n instana-agent
logs-agent:
	kubectl logs --tail=100 -n instana-agent -l app=instana-agent
check-operator:
	kubectl get agent/instana-agent -n instana-agent -o json

create-robotshop-k8s-yamls: ## Create robotshop yamls locally from helm templates
	mkdir ./descriptors
	helm template --name robot-shop --namespace robot-shop  --output-dir ./descriptors helm/

#
# Part 1: Test just catalogue & cart service
#

create-robotshop-namespace: ## Create the robot-shop namespace and switch to it
	kubectl create ns robot-shop
	kubectl config set-context $(shell kubectl config current-context) --namespace=robot-shop
	kubectl create -f resource-quota.yaml --namespace=robot-shop

# https://github.com/instana/robot-shop/blob/master/catalogue/server.js
create-cart-and-catalogue: ## Create the catalog & cart service and backing services mongodb and redis
	kubectl create -f descriptors/robot-shop/templates/catalogue-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/catalogue-service.yaml
	kubectl create -f descriptors/robot-shop/templates/mongodb-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/mongodb-service.yaml
	kubectl create -f descriptors/robot-shop/templates/cart-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/cart-service.yaml
	kubectl create -f descriptors/robot-shop/templates/redis-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/redis-service.yaml

expose-cart: ## Expose the cart to test it
	kubectl expose deployment cart --type=LoadBalancer --name=test-cart-svc

check-status-cart-and-catalogue: ## Some useful commands to check the status of catalogue & cart.
	kubectl get pods,deploy,svc
	@echo "\n---\n"
	kubectl logs -l service=catalogue
	@echo "\n---\n"
	kubectl logs -l service=mongodb
	@echo "\n---\n"
	kubectl logs -l service=cart
	@echo "\n---\n"
	kubectl logs -l service=redis
	@echo "\n---\n"

curl-test-cart: EXTERNAL_CART_IP=$(shell kubectl get svc test-cart-svc -o json | jq -r .status.loadBalancer.ingress[].ip)## Curl the cart service 100 times
curl-test-cart:
	for i in {1..100}; do curl $(EXTERNAL_CART_IP):8080/add/1/HAL-1/1; done

delete-test-cart: ## Delete the exposed cart service.
	kubectl delete svc test-cart-svc


#
# Part 2: Create other services
#

create-other: create-user create-shipping create-payment create-ratings create-web create-dispatch ## Create other services

# [User service](../user/server.js)
# mongo & redis already deployed  
create-user:
	kubectl create -f descriptors/robot-shop/templates/user-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/user-service.yaml

# [Shipping service](../shipping/src/main/java/org/steveww/spark/Main.java)
# cart already deployed
create-shipping:
	kubectl create -f descriptors/robot-shop/templates/shipping-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/shipping-service.yaml
	kubectl create -f descriptors/robot-shop/templates/mysql-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/mysql-service.yaml

# [Payment service](../payment/payment.py)
# cart & user already deployed above
create-payment:
	kubectl create -f descriptors/robot-shop/templates/payment-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/payment-service.yaml
	kubectl create -f descriptors/robot-shop/templates/rabbitmq-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/rabbitmq-service.yaml

# [Ratings service](../ratings/html/api.php)
# mysql already deployed above
create-ratings:
	kubectl create -f descriptors/robot-shop/templates/ratings-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/ratings-service.yaml

create-dispatch:
	kubectl create -f descriptors/robot-shop/templates/dispatch-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/dispatch-service.yaml

# [Web service](../web/default.conf.template)
create-web:
	kubectl create -f descriptors/robot-shop/templates/web-deployment.yaml
	kubectl create -f descriptors/robot-shop/templates/web-service.yaml

#
# Part 3 Testing
#

open-robot-shop: ROBOTSHOP_IP=$(shell kubectl get svc web -o json | jq -r .status.loadBalancer.ingress[].ip)## Open the robot shop in the browser
open-robot-shop: 
	open http://$(ROBOTSHOP_IP):8080

run-load-gen: ROBOTSHOP_IP=$(shell kubectl get svc web -o json | jq -r .status.loadBalancer.ingress[].ip)## Run a load generator
run-load-gen:
	-kubectl create ns robot-shop-load
	kubectl run --generator=deployment/apps.v1 --env HOST=http://$(ROBOTSHOP_IP):8080 --env NUM_CLIENTS=1 --env RUN_TIME=10m loadgen --image robotshop/rs-load -n robot-shop-load

#
# Appendix
#

help: ## Shows this help message
ifeq ($(INSTANA_AGENT_ZONE), INSTANA_AGENT_ZONE)
	@echo "Warning: It seems you haven't configured INSTANA_AGENT_ZONE in the Makefile."
endif
	@echo "See Makefile for details. Available make commands are:"
	@grep -E '^[0-9a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'
