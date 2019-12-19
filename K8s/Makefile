#! /usr/bin/make -f

.DEFAULT_GOAL := help

HUMIO_TOKEN := HUMIO_TOKEN
INSTANA_AGENT_KEY := INSTANA_AGENT_KEY
INSTANA_AGENT_ENDPOINT := INSTANA_AGENT_ENDPOINT
INSTANA_AGENT_ENDPOINT_PORT := INSTANA_AGENT_ENDPOINT_PORT

#
# Part 0: Setup
#

init-helm:
	kubectl -n kube-system create serviceaccount tiller
	kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
	helm init --service-account tiller

install-humio:
	helm repo add humio https://humio.github.io/humio-helm-charts
	helm repo update
	helm install -f humio/humio-agent.yaml humio/humio-helm-charts --set humio-fluentbit.token=$(HUMIO_TOKEN) --name humio --namespace logging

delete-humio:
	helm del --purge humio
	kubectl delete ns logging

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


#
# Part 1: Test just catalogue & cart service
#

create-robotshop-namespace: ## Create the robot-shop namespace and switch to it
	kubectl create ns robot-shop
	kubectl config set-context $(shell kubectl config current-context) --namespace=robot-shop
	kubectl create -f resource-quota.yaml --namespace=robot-shop

# https://github.com/instana/robot-shop/blob/master/catalogue/server.js
create-cart-catalogue: ## Create the catalog & cart service and backing services mongodb and redis
	kubectl create -f descriptors/catalogue-deployment.yaml
	kubectl create -f descriptors/catalogue-service.yaml
	kubectl create -f descriptors/mongodb-deployment.yaml
	kubectl create -f descriptors/mongodb-service.yaml
	kubectl create -f descriptors/cart-deployment.yaml
	kubectl create -f descriptors/cart-service.yaml
	kubectl create -f descriptors/redis-deployment.yaml
	kubectl create -f descriptors/redis-service.yaml

expose-cart: ## Expose the cart to test it
	kubectl expose deployment cart --type=LoadBalancer --name=test-cart-svc

curl-test-cart: EXTERNAL_CART_IP=$(shell kubectl get svc test-cart-svc -o json | jq -r .status.loadBalancer.ingress[].ip)## Curl the cart service 100 times
curl-test-cart:
	for i in {1..100}; do curl $(EXTERNAL_CART_IP):8080/add/1/HAL-1/1; done

delete-test-cart: ## Delete the exposed cart service.
	kubectl delete svc test-cart-svc


#
# Part 2: Create other services
#

create-other: create-user create-shipping create-payment create-ratings create-web ## Create user, shipping, payment, rating, web service and backing services

# [User service](../user/server.js)
# mongo & redis already deployed  
create-user:
	kubectl create -f descriptors/user-deployment.yaml
	kubectl create -f descriptors/user-service.yaml

# [Shipping service](../shipping/src/main/java/org/steveww/spark/Main.java)
# cart already deployed
create-shipping:
	kubectl create -f descriptors/shipping-deployment.yaml
	kubectl create -f descriptors/shipping-service.yaml
	kubectl create -f descriptors/mysql-deployment.yaml
	kubectl create -f descriptors/mysql-service.yaml

# [Payment service](../payment/payment.py)
# cart & user already deployed above
create-payment:
	kubectl create -f descriptors/payment-deployment.yaml
	kubectl create -f descriptors/payment-service.yaml
	kubectl create -f descriptors/rabbitmq-deployment.yaml
	kubectl create -f descriptors/rabbitmq-service.yaml

# [Ratings service](../ratings/html/api.php)
# mysql already deployed above
create-ratings:
	kubectl create -f descriptors/ratings-deployment.yaml
	kubectl create -f descriptors/ratings-service.yaml

# [Web service](../web/default.conf.template)
create-web:
	kubectl create -f descriptors/web-deployment.yaml
	kubectl create -f descriptors/web-service.yaml

open-robot-shop: ROBOTSHOP_IP=$(shell kubectl get svc web -o json | jq -r .status.loadBalancer.ingress[].ip)## Open the robot shop in the browser
open-robot-shop: 
	open http://$(ROBOTSHOP_IP)

run-load-gen: ROBOTSHOP_IP=$(shell kubectl get svc web -o json | jq -r .status.loadBalancer.ingress[].ip)## Run a load generator
run-load-gen:
	-kubectl create ns robot-shop-load
	kubectl run --env HOST=http://$(ROBOTSHOP_IP) --env NUM_CLIENTS=3 loadgen --image robotshop/rs-load -n robot-shop-load

run-artifical-cart-scaler: ## Runs an artifical cart scaler every 4 min
	kubectl apply -f descriptors/cart-scaler.yaml

help: ## Shows this help message
ifeq ($(HUMIO_TOKEN), HUMIO_TOKEN)
	@echo "Warning: It seems you haven't configured HUMIO_TOKEN in the Makefile."
endif
	@echo "See Makefile for details. Available make commands are:"
	@grep -E '^[0-9a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'
