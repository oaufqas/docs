```yml
# Сборка на отдельном сервере с DinD, push в registry
# Деплой на целевой сервер с раннером (общий Docker socket)


stages:
  - build
  - deploy

stage_build:
  tags:
    - virtualKali
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
  script:
    - docker build -t $IMAGE_NAME .
    - docker push $IMAGE_NAME
  
stage_deploy:
  tags:
    - server
  stage: deploy
  image: docker:latest
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
  script:
    - docker stop $CONTAINER_NAME || true
    - docker rm $CONTAINER_NAME || true

    - docker pull $IMAGE_NAME
    - docker run --name $CONTAINER_NAME -e PORT=$PORT -d -p $PORT:$PORT $IMAGE_NAME
      
```
