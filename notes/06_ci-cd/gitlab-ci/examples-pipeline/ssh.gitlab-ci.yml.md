```yml
# Сборка на отдельном сервере с DinD, push в registry
# Деплой на целевой сервер с того же раннера по ssh и pull образов с registry

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
    - virtualKali
  stage: deploy
  before_script:
    - apk add --no-cache openssh-client curl # Установка на alpine

    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa # Добавляем ПРИВАТНЫЙ ключ (публичный уже должен быть на сервере). Можно использовать ЛЮБУЮ пару, созданную на любом устройстве
    - chmod 600 ~/.ssh/id_rsa # Меняем права, чтобы ssh не ругался

    - ssh-keyscan -H $DEPLOY_HOST >> ~/.ssh/known_hosts 2>/dev/null # Добавляем ip сервера, чтобы не спрашивало подтверждение

  script: 
    - |
      ssh $DEPLOY_USER@$DEPLOY_HOST "
        set -e "остановка при ошибке"
        
        docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
        docker stop $CONTAINER_NAME 2>/dev/null || true
        docker rm $CONTAINER_NAME 2>/dev/null || true
        docker pull $IMAGE_NAME
        docker run --name $CONTAINER_NAME -e PORT=$PORT -d -p $PORT:$PORT $IMAGE_NAME"
```