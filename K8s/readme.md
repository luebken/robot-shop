# Kubernetes Setup

A step-by-step guide on how to get the Robot-Shop running in Kubernetes.

The Robot-Shop has the following services:


* web (nginx): [service](./descriptors/web-service.yaml), [deployment](./descriptors/web-deployment.yaml)
    * -> catalogue (nodejs): [service](./descriptors/catalogue-service.yaml), [deployment](./descriptors/catalogue-deployment.yaml)
        * -> mongodb: [service](./descriptors/mongodb-service.yaml), [deployment](./descriptors/mongodb-deployment.yaml)
    * -> user (nodejs) [service](./descriptors/user-service.yaml), [deployment](./descriptors/user-deployment.yaml)
        * -> mongodb: [](./descriptors/mongodb-service.yaml), [](./descriptors/mongodb-deployment.yaml)
        * -> redis: [](./descriptors/redis-service.yaml), [](./descriptors/redis-deployment.yaml)
    * -> cart (nodejs): [](./descriptors/cart-service.yaml), [](./descriptors/cart-deployment.yaml)
        * -> redis: [](./descriptors/redis-service.yaml), [](./descriptors/redis-deployment.yaml)
        * -> catalgue: [](./descriptors/catalogue-service.yaml), [](./descriptors/catalogue-deployment.yaml)
            * -> ...  
    * -> shipping (java): [](./descriptors/shipping-service.yaml), [](./descriptors/shipping-deployment.yaml)
        * -> mysql: [](./descriptors/mysql-service.yaml), [](./descriptors/mysql-deployment.yaml)
        * -> cart: [](./descriptors/cart-service.yaml), [](./descriptors/cart-deployment.yaml)
            * -> ...
    * -> payment (python): [](./descriptors/payment-service.yaml), [](./descriptors/payment-deployment.yaml)
        * -> rabbitmq: [](./descriptors/rabbitmq-service.yaml), [](./descriptors/rabbitmq-deployment.yaml)
        * -> cart: [](./descriptors/cart-service.yaml), [](./descriptors/cart-deployment.yaml)
            * -> ...
        * -> user  [](./descriptors/user-service.yaml), [](./descriptors/user-deployment.yaml)
            * -> ...
    * -> ratings (php) [](./descriptors/ratings-service.yaml), [](./descriptors/ratings-deployment.yaml)
        * -> mysql [](./descriptors/mysql-service.yaml), [](./descriptors/mysql-deployment.yaml)



## Namespace

    # create a dedicated namespace
    $ kubectl create ns robot-shop

## Testing Services

Services are usually not exposed outside of the cluster. For a simple end-to-end test you can temporarily expose that service and test it with curl:

    # expose cart service
    $ kubectl expose deployment cart --type=LoadBalancer --name=test-cart-svc -n robot-shop

    # get the external endpoint IP
    $ kubectl get svc test-cart-svc -n robot-shop

    # test the endoint
    $ curl <EXTERNAL-IP>:8080/health

    # delete the test service
    $ kubectl delete svc test-cart-svc -n robot-shop

As an alternative you can forward a service port locally e.g.:
    $ kubectl port-forward svc/test-cart-svc 8080:8080 -n robot-shop

## Cart

    # create cart deploy & service & redis backend
    # https://github.com/instana/robot-shop/blob/master/cart/server.js
    $ kubectl create -f descriptors/cart-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/cart-service.yaml -n robot-shop
    $ kubectl create -f descriptors/redis-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/redis-service.yaml -n robot-shop

    # check deployment 
    $ kubectl get deploy,po,svc -n robot-shop

    # test cart service
    $ curl <EXTERNAL-CART-IP>:8080/health

## Catalogue

    # create catalogue deployment & service & mongodb backend
    # https://github.com/instana/robot-shop/blob/master/catalogue/server.js
    $ kubectl create -f descriptors/catalogue-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/catalogue-service.yaml -n robot-shop
    $ kubectl create -f descriptors/mongodb-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/mongodb-service.yaml -n robot-shop

    # test catalogue service
    $ curl <EXTERNAL-CATALOGUE-IP>:8080/health
    $ curl <EXTERNAL-CATALOGUE-IP>:8080/products

    # test cart service
    $ curl <EXTERNAL-CART-IP>:8080/add/1/HAL-1/1
    $ curl <EXTERNAL-CART-IP>:8080/cart/1/

## User

    # file://../../user/server.js
    $ kubectl create -f descriptors/user-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/user-service.yaml -n robot-shop
    # mongo & redis already deployed above
    # test user service
    $ curl <EXTERNAL-USER-IP>:8080/health

## Shipping

    # 
    $ kubectl create -f descriptors/shipping-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/shipping-service.yaml -n robot-shop
    $ kubectl create -f descriptors/mysql-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/mysql-service.yaml -n robot-shop
    # cart already deployed above
    # test shipping service
    $ curl <EXTERNAL-SHIPPING-IP>:8080/health

## Payment

    $ kubectl create -f descriptors/payment-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/payment-service.yaml -n robot-shop
    # cart & user already deployed above
    $ kubectl create -f descriptors/rabbitmq-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/rabbitmq-service.yaml -n robot-shop
    $ curl <EXTERNAL-PAYMENT-IP>:8080/health

## Ratings
    $ kubectl create -f descriptors/ratings-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/ratings-service.yaml -n robot-shop
    # mysql already deployed above
    $ curl <EXTERNAL-RATINGS-IP>:8080/health


## Web

    # https://github.com/instana/robot-shop/tree/master/web
    $ kubectl create -f descriptors/web-deployment.yaml -n robot-shop
    $ kubectl create -f descriptors/web-service.yaml -n robot-shop


## Create Load

    # create a dedicated namespace
    $ kubectl create ns robot-shop-load
    $ kubectl run --env HOST=http://35.188.69.251:80 --env NUM_CLIENTS=3 loadgen --image robotshop/rs-load -n robot-shop-load

## Scaling

    # artificial scaling
    kubectl apply -f K8s/descriptors/cart-scaler.yaml -n robot-shop
