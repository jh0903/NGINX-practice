### NGINX

요청에 맞는 정적 파일을 응답해주는 HTTP 웹 서버

reverse proxy server로 활용하여 WAS 서버의 부하를 줄여줄 수 있는 로드 밸런서의 역할도 한다.

💡로드밸런서: 서버에 가해지는 트래픽을 여러대의 서버에게 균등하게 분산시켜주는 역할을 하는 것

## Nginx 를 사용해보고 Nginx 로 Reverse Proxy (로드 밸런서) 만들기

mysite라는 프로젝트를 생성한 후 server디렉토리를 만들고,

server 디렉토리에서 `npm init`과 `npm install express`를 통해 node application을 만든다.

index.js를 다음과 같이 작성한다.

```jsx
const express = require("express")

const app = express()

app.get("/", (req, res) => {
    res.send("I am a endpoint")
})

app.listen(7777, () => {
    console.log("listening on port 7777")
})
```

그리고 다음과 같이 dockerfile을 작성한다.

```docker
#베이직 이미지
FROM node:16 

# 앱 디렉터리 생성
WORKDIR /usr/src/app

# 앱 의존성 설치
# 가능한 경우(npm@5+) package.json과 package-lock.json을 모두 복사하기 위해
# 와일드카드를 사용
COPY package*.json ./

RUN npm install
# 프로덕션을 위한 코드를 빌드하는 경우
# RUN npm ci --omit=dev

# 앱 소스 추가
COPY . .

EXPOSE 7777
CMD [ "npm", "run", "start" ]
```

docker build . -t myserver 를 통해 이미지를 빌드한 후

4개의 컨테이너를 다음 명령어를 통해 만든다.

`docker run -p 1111:7777 -d myserver`

…

`docker run -p 4444:7777 -d myserver`

localhost:8080으로 요청 시 4개의 컨테이너 중 하나로 접속시키기 위해,

nginx를 [설치](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)한 후 기본 설정 파일인 nginx.conf 파일에서 설정을 진행한다.

```bash
http {
    
    include mime.types;

    upstream backendserver {
        server 127.0.0.1:1111;
        server 127.0.0.1:2222;
        server 127.0.0.1:3333;
        server 127.0.0.1:4444;
    }

    server {
        listen 8080;
				
        # /로 시작하는 경로로 접근 시, backendserver로 돌려준다.
        location / {
            proxy_pass http://backendserver/;
        }
    }
}
```

`upstream` : NGINX가 받아들인 요청을 어떤 서버로 흘려보내 줄 것인지 결정할 때 사용됨. default는 라운드로빈 알고리즘

`listen`: 해당 포트로 들어오는 요청을 해당 server {} 블록의 내용에 맞게 처리하겠다는 것을 뜻함

### HTTPS 적용하기

![untitled](https://ko.linux-console.net/common-images/how-to-set-up-nginx-load-balancing-with-ssl-termination/nginx_ssl.png)

서버가 여러개면 각각을 다 암호화/복호화해야 하지만, Nginx를 두면 Nginx에서만 HTTPS를 적용하면 된다.

openSSL을 설정해주기 위해, openSSL for Window를 다운로드받고 환경변수를 설정해준다.

private, public key와 CSR(인증 서명 요청)서, CA 키, CA의 CSR 생성한 후, 인증서(CRT)를 발급받는다. ([참고](https://narup.tistory.com/239))

생성된 인증서와 개인키를 conf 폴더에 넣고 nginx.conf 파일에 다음과 같이 작성한다.

```bash
http {
    
    include mime.types;

    upstream backendserver {
        server 127.0.0.1:1111;
        server 127.0.0.1:2222;
        server 127.0.0.1:3333;
        server 127.0.0.1:4444;
    }

    server {
        listen 8081;
        return 301 https://$host$request_uri;
    }
    
    server {
        listen 443 ssl;

        ssl_certificate jhlee.crt; # 생성된 인증서
        ssl_certificate_key jhlee.key; # 생성된 개인키

        location / {
            proxy_pass http://backendserver/;
        }
    }
}
```
8081포트로 http접속을 하면 443포트로 redirect한다. 443포트로 접속하면 proxy_pass를 통해 upstream 서버인 backendserver로 요청을 보낸다.

→ http://localhost:8081로 접속하면 ssl이 적용된 채 이동하게 된다.

![Untitled](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fc18nV0%2FbtsvOCgCKWs%2F26Q8uCWPwZuIK3gF9BjyRK%2Fimg.png)
