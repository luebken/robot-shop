#! /usr/bin/make -f

.DEFAULT_GOAL := help

INSTANA_AGENT_ZONE := INSTANA_AGENT_ZONE
INSTANA_AGENT_KEY := INSTANA_AGENT_KEY
INSTANA_AGENT_ENDPOINT := INSTANA_AGENT_ENDPOINT
INSTANA_AGENT_ENDPOINT_PORT := INSTANA_AGENT_ENDPOINT_PORT

#
# Part 0: Setup
#

create-k8s-yamls: ## Create yamls from helm templates
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
# Appendix
#

help: ## Shows this help message
ifeq ($(INSTANA_AGENT_ZONE), INSTANA_AGENT_ZONE)
	@echo "Warning: It seems you haven't configured INSTANA_AGENT_ZONE in the Makefile."
endif
	@echo "See Makefile for details. Available make commands are:"
	@grep -E '^[0-9a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'
