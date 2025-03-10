---
title: Amplify에서 self-hosting으로 서비스 전환
date: 2025-03-11 00:22:15 +09:00
categories: [dev, etc]
tags: [dev, etc] 
---

## Amplify를 포기한 이유

amplify에서 이전 했던건 작년 3월쯤이였던것 같다  
aws cognito 기반의 로그인을 next auth로 이전하는 작업을 하는데  
특정 관리되는 해더의 내용을 읽을 수 없는 이슈가 있어 이전하기로 결정하였다.

## 어디로 이전할까?

첫 후보는 Vercel 에서 돌리는거였다.  
next.js 을 개발하는 곳이다보니 가장 안정정이라는 장점이 있었지만  
지금만 해도 12만 MAU에 160만 PV 나와 비용 이슈로 기각 되었다.

다음은 Azure에 있는 관리형 서비스를 사용하는거였다.  
aws 버리고 Azure로 간다는게 좀 별로긴 했는데 지금은 만료되었지만 적지 않은 양의 크레딧이 있었다.  

![azure_credit](/assets/img/post/dev/etc/amplify-to-selfhost/azure_credit.png)

그래서 Azure Static Web Apps를 이용해서 ci/cd를 만들었는데 정확히는 생각 안나지만 뭔가 이슈가 있어서 포기하였다.  
지금도 ssr 지원하는 하이브리드 Next.js 배포는 미리 보기 상태인걸 보면 포기하길 잘한 것 같다  

결국 마지막 후보로 직접 vm 인스턴스 파서 호스팅 하는걸로 결정되었다.

## ci/cd 구협

## docker compose
다 사용 못할정도의 크레딧이 있지만 당장은 비용 감축을 고려하고 만들어야 하기 때문에 단일 vm에 docker compose 이용해서 올리기로 하였다.  

지금은 단일이지만 혹시 나중에 다중 vm으로 전환하고 로드밸런서 사용할것까지 고려해서 구조를 잡았다.

```yml
name: mana-client
services:
  nextjs_blue:
    image: ${ACR_NAME}/${IMAGE_NAME}:${IMAGE_TAG}
    environment:
      ...
    ports:
      - "4000:3000"
    restart: always

  nextjs_green:
    image: ${ACR_NAME}/${IMAGE_NAME}:${IMAGE_TAG}
    environment:
      ...
    ports:
      - "4001:3000"
    restart: always
```
일단 단일 vm에서 무중단 배포를 위해 blue,green 2개를 만들었다  

### nginx config
그다음 여기서 만든 blue,green을 사용하기 위한 리버스 프록시로 nginx를 작성했다.  

```nginx
upstream mana_backend {
  server localhost:4000;
  server localhost:4001 backup;
}

server {
    if ($host = www.mana.so) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    if ($host = mana.so) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


  listen 80;
  server_name mana.so *.mana.so;
  return 301 https://$host$request_uri;

}

server {
  listen 443 ssl;
  server_name www.mana.so;

  ssl_certificate ...;
  ssl_certificate_key ...;
  return 301 https://mana.so$request_uri;
}


server {
  listen 443 ssl;
  server_name mana.so;
  ssl_certificate ...;
  ssl_certificate_key ...;

  location / {
    proxy_pass http://mana_backend;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

SSL은 Certbot 이용해서 처리했고 upstream backup을 이용해서 하나의 서버에 접근하지 못하면 다른 서버에서 들고오는 구조이다.

### 쉘 스크립트
그다음은 쉘 스크립트를 이용해서 도커를 받아오고 blue green 전환 하는 코드를 작성했다  

```bash
#!/bin/bash

ACR_NAME=$1
IMAGE_NAME=$2
NEW_IMAGE_TAG=$3
DOCKER_PULL_TOKEN=$4

export ACR_NAME=$ACR_NAME
export IMAGE_NAME=$IMAGE_NAME

BLUE_PORT=4000
GREEN_PORT=4001
NGINX_CONF="/etc/nginx/sites-available/mana.so"

docker login -u mana-client-pull -p $DOCKER_PULL_TOKEN $ACR_NAME

docker pull $ACR_NAME/$IMAGE_NAME:$NEW_IMAGE_TAG

CURRENT_SERVICE=$(docker ps --filter "name=nextjs_" --format "{{.Names}}" | grep -q "nextjs_blue" && echo "blue" || echo "green")
CURRENT_IMAGE_TAG=$(docker ps --filter "name=nextjs_${CURRENT_SERVICE}" --format "{{.Image}}" | cut -d: -f2)

if [ "$CURRENT_SERVICE" == "blue" ]; then
    NEW_SERVICE="green"
    SERVICE_PORT=$GREEN_PORT
else
    NEW_SERVICE="blue"
    SERVICE_PORT=$BLUE_PORT
fi

echo "Current service: $CURRENT_SERVICE:$CURRENT_IMAGE_TAG"
echo "New service: $NEW_SERVICE:$NEW_IMAGE_TAG"

IMAGE_TAG=$NEW_IMAGE_TAG docker compose up -d nextjs_$NEW_SERVICE

for i in {1..30}; do
    if curl -s http://localhost:${SERVICE_PORT}/api/ping > /dev/null; then
        echo "New service is up and running"
        break
    fi
    if [ $i -eq 30 ]; then
        echo "New service failed to start"
        exit 1
    fi
    sleep 1
done

echo "Updating Nginx configuration..."
if [ "$NEW_SERVICE" == "blue" ]; then
    sudo sed -i "s/server localhost:$GREEN_PORT;/server localhost:$BLUE_PORT;/" $NGINX_CONF
    sudo sed -i "s/server localhost:$BLUE_PORT backup;/server localhost:$GREEN_PORT backup;/" $NGINX_CONF
else
    sudo sed -i "s/server localhost:$BLUE_PORT;/server localhost:$GREEN_PORT;/" $NGINX_CONF
    sudo sed -i "s/server localhost:$GREEN_PORT backup;/server localhost:$BLUE_PORT backup;/" $NGINX_CONF
fi

sudo nginx -t
sudo systemctl reload nginx

IMAGE_TAG=$CURRENT_IMAGE_TAG docker compose stop nextjs_$CURRENT_SERVICE

docker system prune -af
```

간단하게 설명하면 `docker ps`로 기존에 돌아가는게 blue인지 green인지 확인하고  
새로은 컨테이너를 돌리는것이다.  
컨테이너 실행이 확인되면 `sed`로 새로 돌아가는 컨테이너가 메인이 되도록 컨피그 파일을 수정하고 기존에 돌아가던 도커를 종료시킨다.  

### github action
마지막으로 이 모든걸 돌릴 github action을 만들면 완성

```yml
name: Deploy to Azure

on:
  push:
    branches:
      - master

env:
  ACR_NAME: ...
  IMAGE_NAME: mana-client

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Login to ACR for pull
        uses: docker/login-action@v1
        with:
          registry: ${{ env.ACR_NAME }}
          username: mana-client-pull
          password: ${{ secrets.DOCKER_PULL_TOKEN }}

      - name: Login to ACR for push
        uses: docker/login-action@v1
        with:
          registry: ${{ env.ACR_NAME }}
          username: mana-client-push
          password: ${{ secrets.DOCKER_PUSH_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v2
        with:
          context: .
          file: ./scripts/deploy/Dockerfile
          push: true
          tags: |
            ${{ env.ACR_NAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.ACR_NAME }}/${{ env.IMAGE_NAME }}:${{ github.run_number }}.${{ github.run_attempt }}
            ${{ env.ACR_NAME }}/${{ env.IMAGE_NAME }}:latest
      
      - name: Upload deployment files
        uses: actions/upload-artifact@v4
        with:
          name: deploy-files
          path: |
            scripts/deploy/docker-compose.yml
            scripts/deploy/deploy.sh

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    strategy:
      matrix:
        server: [SERVER_1]  # 배포할 서버 목록
    steps:
      - name: Download deployment files
        uses: actions/download-artifact@v4
        with:
          name: deploy-files
          path: deploy-files

      - name: Copy deployment files to server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets[format('HOST_{0}', matrix.server)] }}
          username: mana
          key: ${{ secrets[format('SSH_KEY_{0}', matrix.server)] }}
          port: 22
          source: "deploy-files/*"
          target: "/home/mana/app/mana-client-docker/${{ github.run_number }}.${{ github.run_attempt }}"
          strip_components: 1

      - name: Deploy to VM
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets[format('HOST_{0}', matrix.server)] }}
          username: mana
          key: ${{ secrets[format('SSH_KEY_{0}', matrix.server)] }}
          port: 22
          script: |
            cd /home/ubuntu/app/mana-client-docker/${{ github.run_number }}.${{ github.run_attempt }}
            chmod +x deploy.sh
            ./deploy.sh ${{ env.ACR_NAME }} ${{ env.IMAGE_NAME }} ${{ github.run_number }}.${{ github.run_attempt }} ${{ secrets.DOCKER_PULL_TOKEN }}

      - name: Notify deployment status
        if: always()
        uses: tsickert/discord-webhook@v5.3.0
        with:
          webhook-url: ...
          username: github-build
          avatar-url: https://github.githubassets.com/assets/GitHub-Mark-ea2971cee799.png
          content: "Deployment to ${{ matrix.server }} ${{ job.status == 'success' && 'succeeded' || 'failed' }}!"
          embed-title: "Deployment Status"
          embed-description: |
            Repository: ${{ github.repository }}
            Commit: ${{ github.sha }}
            Branch: ${{ github.ref }}
            Author: ${{ github.actor }}
            Status: ${{ job.status }}
          embed-color: ${{ job.status == 'success' && '65280' || '16711680' }}
```
나중에 다른 서버를 추가할 수 있도록 matrix를 이용했다.  

## 한계점 및 고려사항

위에서도 말했지만 이런 구조에는 각종 한계점이 있다.

### 1. 단일 vm 환경

비용 회적화를 생각해서 단일vm으로 구축했지만 나중에 서비스가 더욱 성장하거나
안정성이 중요하면 고가용성 보장을 위해 다중 vm에 로드밸런서 사용이 필요하다.

### 2. 확장성 제한

큰 트래픽이 오는 상황에 즉각적인 스케일링이 어렵다는 단점이 있긴 하나  
지금 서비스는 1차적으로 cdn을 통해 전달되고 브라우저단 캐싱도 있어  
순식간에 대규모 트래픽이 오는 상황 아니면 수직적 또는 수평적 확장으로 감당 가능하다고 생각했다.  

### 3. 관리 부담

기존은 클라우드에서 관리되는 서비스를 이용하여 구축하였지만  
지금은 vm 인스턴스을 이용해서 구축하였으므로 관리해야 할 점이 늘어난건 사실이다.  

일단은 vm의 cpu, 메모리, 내트워크 트래픽등을 자동으로 모니터링 해서 이상 발생시 경고가 올 수 있도록 하였다.  
추가로 혹시모를 vm 종료에 대비하기 위해 특정 시간마다 서버에 지속적으로 요청을 전달해 서버 중단을 대응하고 있다.  

## 결론

클라우드 서비스의 편리함에도 불구하고 지금처럼 특정 요구사항  
(amplify에서 특정 헤더가 전달되지 않음)이 있을 경우에는 직접 구축하는게 적합할 수 있다.  

물론 이런 구조가 고가용성이나 즉각적인 스케일링 측면에서는 한계가 있지만, 소규모 스타트업이나 중소 규모 서비스에서는 비용 효율성과 충분한 기능성을 제공할 수 있었다.

앞으로 서비스가 성장해서 규모가 커지면 다중 VM 환경이나 k8s같은 더 복잡한 인프라로 전환 할 수도 있지만, 지금 이 시점에서는 개발 속도, 비용 효율성, 그리고 필요한 기능성을 모두 충족시키는 적절한 선택이었다고 생각한다.
