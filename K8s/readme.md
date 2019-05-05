# Kubernetes Setup

A step-by-step guide on how to get the Robot-Shop running in Kubernetes.

## Prerequisites

### Instana Agent

If not done already install the Instana agent:

    $ helm init --service-account tiller
    $ helm install --name instana-agent --namespace instana-agent --set agent.key=INSTANA_AGENT_KEY --set zone.name=CLUSTER_NAME

## Service Overview

The Robot-Shop has the following Kubernetes services and deployments defined in the [descriptors](../descriptors/) directory:


* web (nginx): [web-service.yaml](./descriptors/web-service.yaml), [web-deployment.yaml](./descriptors/web-deployment.yaml)
    * -> catalogue (nodejs): [catalogue-service.yaml](./descriptors/catalogue-service.yaml), [catalogue-deployment.yaml](./descriptors/catalogue-deployment.yaml)
        * -> mongodb: [mongodb-service.yaml](./descriptors/mongodb-service.yaml), [mongodb-deployment.yaml](./descriptors/mongodb-deployment.yaml)
    * -> user (nodejs) [user-service.yaml](./descriptors/user-service.yaml), [user-deployment.yaml](./descriptors/user-deployment.yaml)
        * -> mongodb: [mongodb-service.yaml](./descriptors/mongodb-service.yaml), [mongodb-deployment.yaml](./descriptors/mongodb-deployment.yaml)
        * -> redis: [redis-service.yaml](./descriptors/redis-service.yaml), [redis-deployment.yaml](./descriptors/redis-deployment.yaml)
    * -> cart (nodejs): [cart-service.yaml](./descriptors/cart-service.yaml), [cart-deployment.yaml](./descriptors/cart-deployment.yaml)
        * -> redis: [redis-service.yaml](./descriptors/redis-service.yaml), [redis-deployment.yaml](./descriptors/redis-deployment.yaml)
        * -> catalgue: [catalogue-service.yaml](./descriptors/catalogue-service.yaml), [catalogue-deployment.yaml](./descriptors/catalogue-deployment.yaml)
            * -> ...  
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

## Namespace

    # create a dedicated namespace
    $ kubectl create ns robot-shop
    # switch to that namaspace for kubectl
    $ kubectl config set-context --current --namespace=robot-shop

## Testing Services

Services are usually not exposed outside of the cluster. For a simple end-to-end test you can temporarily expose that service and test it with curl:

    # expose cart service
    $ kubectl expose deployment cart --type=LoadBalancer --name=test-cart-svc

    # get the external endpoint IP
    $ kubectl get svc test-cart-svc

    # test the endoint
    $ curl <EXTERNAL-IP>:8080/health
    $ for i in {1..100}; do curl 35.222.143.6:8080/; done

    # delete the test service
    $ kubectl delete svc test-cart-svc

As an alternative you can forward a service port locally e.g.:
    $ kubectl port-forward svc/test-cart-svc 8080:8080

## Catalogue & Cart

Let's tart by creatng the catalogue & cart service with their respective backend services:

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

To test this we need to expose a service:

    $ kubectl expose deployment cart --type=LoadBalancer --name=test-cart-svc
    $ EXTERNAL_CART_IP=$(kubectl get svc test-cart-svc -o json | jq -r .status.loadBalancer.ingress[].ip)
    $ curl $EXTERNAL_CART_IP:8080/add/1/HAL-1/1
    $ curl $EXTERNAL_CART_IP:8080/cart/1/

    # create some trivial load
    $ for i in {1..100}; do curl $EXTERNAL_CART_IP:8080/add/1/HAL-1/1; done

## User

    # file://../../user/server.js
    $ kubectl create -f descriptors/user-deployment.yaml
    $ kubectl create -f descriptors/user-service.yaml
    # mongo & redis already deployed above
    # test user service
    $ curl <EXTERNAL-USER-IP>:8080/health

## Shipping

    # 
    $ kubectl create -f descriptors/shipping-deployment.yaml
    $ kubectl create -f descriptors/shipping-service.yaml
    $ kubectl create -f descriptors/mysql-deployment.yaml
    $ kubectl create -f descriptors/mysql-service.yaml
    # cart already deployed above
    # test shipping service
    $ curl <EXTERNAL-SHIPPING-IP>:8080/health

## Payment

    $ kubectl create -f descriptors/payment-deployment.yaml
    $ kubectl create -f descriptors/payment-service.yaml
    # cart & user already deployed above
    $ kubectl create -f descriptors/rabbitmq-deployment.yaml
    $ kubectl create -f descriptors/rabbitmq-service.yaml
    $ curl <EXTERNAL-PAYMENT-IP>:8080/health

## Ratings
    $ kubectl create -f descriptors/ratings-deployment.yaml
    $ kubectl create -f descriptors/ratings-service.yaml
    # mysql already deployed above
    $ curl <EXTERNAL-RATINGS-IP>:8080/health


## Web

    # https://github.com/instana/robot-shop/tree/master/web
    $ kubectl create -f descriptors/web-deployment.yaml
    $ kubectl create -f descriptors/web-service.yaml


## Create Load

    # create a dedicated namespace
    $ kubectl create ns robot-shop-load
    $ kubectl run --env HOST=http://35.188.69.251:80 --env NUM_CLIENTS=3 loadgen --image robotshop/rs-load-load

## Scaling

    # artificial scaling
    kubectl apply -f K8s/descriptors/cart-scaler.yaml
