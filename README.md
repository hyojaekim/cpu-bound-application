# CPU Bound Application

## I/O Burst vs CPU Burst, I/O Bound vs CPU Bound

- **I/O Burst** : I/O를 하는 시간을 의미하며, I/O Burst가 많은 것을 **I/O Bound Application**
- **CPU Burst** : CPU를 사용하는 시간을 의미하며, CPU Burst가 많은 것을 **CPU Bound Application**

## Artillery로 성능 측정하기

### 시작하기

```bash
npm install -g artillery #Artillery 설치하기
artillery run --output report.json cpu-test.yaml #테스트 실행 -> json 파일 생성
artillery report ./report.json #json 파일 -> html로 변환해서 실행하기
```

### Artillery 스크립트

```bash
config:
  target: "http://34.64.213.244"
  phases:
    - duration: 60
      arrivalRate: 2
      name: Warm up
scenarios:
  # We define one scenario:
  - name: "Just get hash"
    flow:
      # Get the details of the product:
      - get:
          url: "/hash/123"
```

### Scale-up 후, 성능 테스트

![Scale up 후, 테스트](./img/scale-up-test.png)

- `e2-micro(vCPU 2개, 1GB 메모리)` -> `e2-highmem-8(vCPU 8개, 64GB 메모리)`
- `duration:360, arrivalRate:2`로 설정 후, 안정적으로 트래픽을 받을 수 있었다.
- 튀는 구간이 있는데 이는 VM을 사용하기 때문에 어쩔 수 없는 것 같다. PM과 달리 VM은 인접한 인스턴스와 자원을 공유하기 때문에 이런 현상이 발생할 수 있다.

### 성능 테스트하기 위한 기준 만들기

- 내가 예상하는 TPS보다 여유롭게 설정
- 기대 Latency를 만족할 때까지 성능 테스트하기
  - 단일 요청 레이턴시가 기대 레이턴시보다 높다면, Scale-out으로 해결되지 않는다. 따라서 코드가 비효율적이나, I/O가 병목이거나, 네트워크에서 레이턴시가 발생할 수 있다. 따라서 다양한 부분들을 고려할 수 있어야 한다.
- Scale-out해도 성능이 늘지 않으면 병목을 의심하자.
- 처음에는 짧은 시간으로 적은 TPS로 시작 -> TPS를 조금씩 올려서 언제부터 짧은 시간조차 버티지 못하는지 확인하기
- 서비스 Peak 시간에 유입되는 트래픽을 버틸 수 있는 수준까지 테스트가 필요하다.
- 짧은 시간 매우 높은 트래픽을 견딜 수 있는 서버 개수나 저장소, 코드 병목을 찾았다면 스트레스 테스트 시간을 아주 길게 테스트해보자. (장기적으로 자원이 고갈되는 상태일 수 있기 때문)

### 그래프 각 선의 의미

- **[max]** 가장 오래 걸린 요청 → 응답시간
- **[p95]** 전체 HTTP 트랜잭션 중 가장 빠른것부터 95%까지 (대부분의 트래픽)
- **[p50]** 전체 HTTP 트랜잭션 중 가장 빠른것부터 50%까지 (절반 트래픽)
- **[min]** 가장 빠르게 온 요청 → 응답시간

### 실제 밴치마크를 잴 때 어떤 것을 주로 사용하는가? 
- 실제 벤치마크를 잴 때 **p99와 p95 두가지 그래프를 많이 그린다.**
  p99는 대부분 트래픽이 해당되기 때문에 실제 애플리케이션 성능에 가깝기 때문이다.
- max는 네트워크는 상당히 불안정한 존재이기 때문에 전체 성능 측정 결과에 영향을 미치는 것이 바람직하지 않다.

## Docker 이미지로 만들어 배포하기

![docker](./img/docker.png)

1. dockerfile을 빌드하면 docker 이미지가 된다. (로컬 환경)
   - Build 명령어 `docker build -t 사용자이름/저장소이름 .`
2. 이미지를 저장소(dockerhub)에 push하면 해당 저장소에 업로드 된다.
   - Push 명령어 `docker push 사용자이름/저장소이름`
3. 저장소에 있는 이미지를 pull하면 이미지가 다운로드 된다. (GCP Instance)
   - Pull 명령어 `docker pull 사용자이름/저장소이름`
4. 다운로드 된 이미지를 run하면 container가 되어서 비로소 애플리케이션이 실행된다.
   - Run 명령어 `docker run -p {HOST_PORT}:{CONTAINER_PORT} 사용자이름/저장소이름`
    
### Jenkins를 통해 Dockerized Application 배포 자동화하기

* Jenkinks 인스턴스, Worker 인스턴스 2개 만들기
* Jenkins에서 개인키와 공개키를 쌍으로 만들어 Worker 인스턴스 메타데이터에 공개키를 넣어준다.
    * 대칭키: 암호화에 사용하는 키와 복호화에 사용하는 키가 같은 것.
    * 비대칭키: 암호화에 사용하는 키와 복호화에 사용하는 키가 다른 것. (다른 값이지만 연관되어 있는 값이다)
* Publish over SSH 작성하기
* 배포 스크립트 작성

### 적용하면서 발생했던 문제들
1. `bash: docker: command not found` - 도커가 설치되지 않아서 발생하는 오류
    * `sudo yum install docker`
    * `sudo systemctl start docker`
2. `/var/run/docker.sock: connect: permission denied` - docker.sock 권한 문제
    * `sudo chmod 666 /var/run/docker.sock`
3. `Jenkins에서 톰캣이 띄워졌지만, 빨간색 그래프나 시계가 계속 도는 현상` - Jenkins에서 배포가 끝나지 않았다고 인식하는 문제
    * 백그라운드로 실행이 필요 `nohup docker run -p 80:8080 사용자이름/저장소이름 > /dev/null 2>^&1 &`
    
## 인스턴스가 n개일 때, 처리 가능한 TPS 비교

### Docker container 우아하게(?) 종료하기

- [graceful shutdown](https://github.com/hyojaekim/TIL/blob/master/2021/2021-02-21.md)
- `docker ps | grep {이미지 이름}` 현재 실행중인 애플리케이션 확인하기
- `docker container kill -s 15 {Container ID}` `-s 15` 옵션으로 우아하게 종료(애플리케이션이 실행중인 일을 모두 처리하고 종료한다.)

### Case 1. 인스턴스가 3개일 때

1. `360s - instance:3, ArrivalRate:1`
   ![stress test 1](./img/stress-test-1.png)
2. `360s - instance:3, ArrivalRate:8`
   ![stress test 2](./img/stress-test-2.png)
3. `360s - instance:3, ArrivalRate:16`
   ![stress test 3](./img/stress-test-3.png)

> ArrivalRate(vuser) 1 ~ 8은 모든 요청에 대해 튀는 구간이 발생하지만, 정상적으로 응답한다.
> ArrivalRate(vuser) 16부터 300ms 까지 정상적으로 응답하다가, 300ms 이후로부터 500, 502 에러가 발생한다.

- [500, 502 차이](https://github.com/hyojaekim/TIL/blob/master/2021/2021-02-21.md)
 
### Case 2. 인스턴스 3개일 때, 2개를 내렸다가 다시 올렸을 때

1. `1번`, `2번`, `3번` 인스턴스가 살아있는지 확인하기 (모든 애플리케이션이 살아있는 상태에서 진행)
2. 스트레스 테스트 진행 (360s - instance:3, ArrivalRate:2)
3. `1번`, `2번` 인스턴스에서 애플리케이션 종료시키기
4. 재배포하기

![stress test 4](./img/stress-test-4.png)

> 중간에 튀는 구간이 2개의 인스턴스를 종료시켰을 때로 예상된다.
> 하지만 모든 요청을 200으로, 정상적인 응답을 받을 수 있다는 것을 볼 수 있다. 
> 2개의 인스턴스를 종료시켰음에도 불구하고 처리할 수 있었던 이유는 정상적으로 무중단 배포가 이루어졌음을 알 수 있다.