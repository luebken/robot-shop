create-set-namespace:
	kubectl create ns robot-shop
    # switch to that namaspace for kubectl
    # kubectl config set-context $(kubectl config current-context) --namespace=robot-shop

# https://github.com/instana/robot-shop/blob/master/catalogue/server.js
create-cart-catalogue:
	kubectl create -f descriptors/catalogue-deployment.yaml
	kubectl create -f descriptors/catalogue-service.yaml
	kubectl create -f descriptors/mongodb-deployment.yaml
	kubectl create -f descriptors/mongodb-service.yaml
	kubectl create -f descriptors/cart-deployment.yaml
	kubectl create -f descriptors/cart-service.yaml
	kubectl create -f descriptors/redis-deployment.yaml
	kubectl create -f descriptors/redis-service.yaml

expose-cart:
	kubectl expose deployment cart --type=LoadBalancer --name=test-cart-svc

test-cart: EXTERNAL_CART_IP=$(shell kubectl get svc test-cart-svc -o json | jq -r .status.loadBalancer.ingress[].ip)
test-cart:
	curl $(EXTERNAL_CART_IP):8080/add/1/HAL-1/1

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

test-robot-shop: ROBOTSHOP_IP=$(shell kubectl get svc web -o json | jq -r .status.loadBalancer.ingress[].ip)
test-robot-shop: 
	open http://$(ROBOTSHOP_IP)

load: ROBOTSHOP_IP=$(shell kubectl get svc web -o json | jq -r .status.loadBalancer.ingress[].ip)
load:
	-kubectl create ns robot-shop-load
	kubectl run --env HOST=http://$(ROBOTSHOP_IP) --env NUM_CLIENTS=3 loadgen --image robotshop/rs-load -n robot-shop-load