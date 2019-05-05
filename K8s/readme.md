# Kubernetes Setup

A step-by-step guide on how to get the Robot-Shop running in Kubernetes.

## Prerequisites

If not done already install the Instana agent:

    $ helm init --service-account tiller
    $ helm install --name instana-agent --namespace instana-agent --set agent.key=INSTANA_AGENT_KEY --set zone.name=CLUSTER_NAME

## Services overview

The Robot-Shop has the following Kubernetes services and deployments defined in the [descriptors](../descriptors/) directory:

* web (nginx): [web-service.yaml](./descriptors/web-service.yaml), [web-deployment.yaml](./descriptors/web-deployment.yaml)
    * -> cart (nodejs): [cart-service.yaml](./descriptors/cart-service.yaml), [cart-deployment.yaml](./descriptors/cart-deployment.yaml)
        * -> redis: [redis-service.yaml](./descriptors/redis-service.yaml), [redis-deployment.yaml](./descriptors/redis-deployment.yaml)
        * -> catalgue: [catalogue-service.yaml](./descriptors/catalogue-service.yaml), [catalogue-deployment.yaml](./descriptors/catalogue-deployment.yaml)
            * -> ...  
    * -> catalogue (nodejs): [catalogue-service.yaml](./descriptors/catalogue-service.yaml), [catalogue-deployment.yaml](./descriptors/catalogue-deployment.yaml)
        * -> mongodb: [mongodb-service.yaml](./descriptors/mongodb-service.yaml), [mongodb-deployment.yaml](./descriptors/mongodb-deployment.yaml)

    * -> user (nodejs) [user-service.yaml](./descriptors/user-service.yaml), [user-deployment.yaml](./descriptors/user-deployment.yaml)
        * -> mongodb: [mongodb-service.yaml](./descriptors/mongodb-service.yaml), [mongodb-deployment.yaml](./descriptors/mongodb-deployment.yaml)
        * -> redis: [redis-service.yaml](./descriptors/redis-service.yaml), [redis-deployment.yaml](./descriptors/redis-deployment.yaml)
    * -> shipping (java): [shipping-service.yaml](./descriptors/shipping-service.yaml), [shipping-deployment.yaml](./descriptors/shipping-deployment.yaml)
        * -> mysql: [mysql-service.yaml](./descriptors/mysql-service.yaml), [mysql-deployment.yaml](./descriptors/mysql-deployment.yaml)
        * -> cart: [cart-service.yaml](./descriptors/cart-service.yaml), [cart-deployment.yaml](./descriptors/cart-deployment.yaml)
            * -> ...
    * -> payment (python): [payment-service.yaml](./descriptors/payment-service.yaml), [web-deployment.yaml](./descriptors/payment-deployment.yaml)
        * -> rabbitmq: [rabbitmq-service.yaml](./descriptors/rabbitmq-service.yaml), [rabbitmq-deployment.yaml](./descriptors/rabbitmq-deployment.yaml)
        * -> cart: [cart-service.yaml](./descriptors/cart-service.yaml), [cart-deployment.yaml](./descriptors/cart-deployment.yaml)
            * -> ...
        * -> user  [user-service.yaml](./descriptors/user-service.yaml), [user-deployment.yaml](./descriptors/user-deployment.yaml)
            * -> ...
    * -> ratings (php) [ratings-service.yaml](./descriptors/ratings-service.yaml), [ratings-deployment.yaml](./descriptors/ratings-deployment.yaml)
        * -> mysql [mysql-service.yaml](./descriptors/mysql-service.yaml), [mysql-deployment.yaml](./descriptors/mysql-deployment.yaml)

## Catalogue & Cart Services

Let's start by creatng the catalogue & cart service with their respective backend services:

    # create a dedicated namespace
    $ kubectl create ns robot-shop
    # switch to that namaspace for kubectl
    $ kubectl config set-context --current --namespace=robot-shop

    # create catalogue deployment & service & mongodb backend
    # https://github.com/instana/robot-shop/blob/master/catalogue/server.js
    $ kubectl create -f descriptors/catalogue-deployment.yaml
    $ kubectl create -f descriptors/catalogue-service.yaml
    $ kubectl create -f descriptors/mongodb-deployment.yaml
    $ kubectl create -f descriptors/mongodb-service.yaml

    # create cart deploy & service & redis backend
    # https://github.com/instana/robot-shop/blob/master/cart/server.js
    $ kubectl create -f descriptors/cart-deployment.yaml
    $ kubectl create -f descriptors/cart-service.yaml
    $ kubectl create -f descriptors/redis-deployment.yaml
    $ kubectl create -f descriptors/redis-service.yaml


## Testing Services

Most microservices are usually not exposed outside of the cluster. For a simple end-to-end test you can temporarily expose that service and test it with curl:

    # expose the cart service
    $ kubectl expose deployment cart --type=LoadBalancer --name=test-cart-svc
    $ EXTERNAL_CART_IP=$(kubectl get svc test-cart-svc -o json | jq -r .status.loadBalancer.ingress[].ip)
    
    # test it by curling
    $ curl $EXTERNAL_CART_IP:8080/add/1/HAL-1/1
    $ curl $EXTERNAL_CART_IP:8080/cart/1/
    
    # testing it by curling it a 100 times
    $ for i in {1..100}; do curl $EXTERNAL_CART_IP:8080/add/1/HAL-1/1; done

    # delete the test service
    $ kubectl delete svc test-cart-svc

As an alternative you can forward a service port locally:

    $ kubectl port-forward svc/cart-svc 8080:8080

Or you get wget it from within the cluster:

    $ kubectl run busybox --image=busybox:1.28 --rm -it --restart=Never --comand wget -qO- http://cart:8080/add/1/HAL-1/1

## User, Shipping, Payment, Ratings and Web-Service

To bring up the whole robot-shop deploy the rest of the services:

    # [User service](../user/server.js)
    # mongo & redis already deployed    
    $ kubectl create -f descriptors/user-deployment.yaml
    $ kubectl create -f descriptors/user-service.yaml

    # [Shipping service](../shipping/src/main/java/org/steveww/spark/Main.java)
    # cart already deployed
    $ kubectl create -f descriptors/shipping-deployment.yaml
    $ kubectl create -f descriptors/shipping-service.yaml
    $ kubectl create -f descriptors/mysql-deployment.yaml
    $ kubectl create -f descriptors/mysql-service.yaml

    # [Payment service](../payment/payment.py)
    # cart & user already deployed above
    $ kubectl create -f descriptors/payment-deployment.yaml
    $ kubectl create -f descriptors/payment-service.yaml
    $ kubectl create -f descriptors/rabbitmq-deployment.yaml
    $ kubectl create -f descriptors/rabbitmq-service.yaml

    # [Ratings service](../ratings/html/api.php)
    # mysql already deployed above
    $ kubectl create -f descriptors/ratings-deployment.yaml
    $ kubectl create -f descriptors/ratings-service.yaml

    # [Web service](../web/default.conf.template)
    $ kubectl create -f descriptors/web-deployment.yaml
    $ kubectl create -f descriptors/web-service.yaml

## Testing the Robotshop

    $ ROBOTSHOP_IP=$(kubectl get svc test-cart-svc -o json | jq -r .status.loadBalancer.ingress[].ip)
    # open http://$ROBOTSHOP_IP

//TODO gif from navigating the robotshop

    # create some artificial load
    $ kubectl create ns robot-shop-load
    $ kubectl run --env HOST=http://35.188.69.251:80 --env NUM_CLIENTS=3 loadgen --image robotshop/rs-load-load

//TODO gif from dependencies in instana

## Scaling

    # artificial scaling
    kubectl apply -f K8s/descriptors/cart-scaler.yaml
