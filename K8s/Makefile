create-set-namespace:
	kubectl create ns robot-shop
    # switch to that namaspace for kubectl
    # kubectl config set-context $(kubectl config current-context) --namespace=robot-shop

# https://github.com/instana/robot-shop/blob/master/catalogue/server.js
create-catalogue:
	kubectl create -f descriptors/catalogue-deployment.yaml
	kubectl create -f descriptors/catalogue-service.yaml
	kubectl create -f descriptors/mongodb-deployment.yaml
	kubectl create -f descriptors/mongodb-service.yaml

create-cart:
	kubectl create -f descriptors/cart-deployment.yaml
	kubectl create -f descriptors/cart-service.yaml
	kubectl create -f descriptors/redis-deployment.yaml
	kubectl create -f descriptors/redis-service.yaml

expose-cart:
	kubectl expose deployment cart --type=LoadBalancer --name=test-cart-svc

# test-cart:
	#EXTERNAL_CART_IP = $(kubectl get svc test-cart-svc -o json | jq -r .status.loadBalancer.ingress[].ip)
	# EXTERNAL_CART_IP := $(echo test)
	# echo EXTERNAL_CART_IP
	#curl $(EXTERNAL_CART_IP):8080/add/1/HAL-1/1
