# 카파 아키텍처 실습
> 벌크 색인 대신에 모든 데이터를 스트리밍 형태로 유지하고, 스트림 파이프라인을 데이터 저장소로 인지하고 이를 통해 기본적인 데이터 처리 및 재처리를 전재하고 있기 때문에 서빙 레이어에서 `MySQL 혹은 MongoDB`같은 동적 업데이트를 지원하는 엔진이 적합합니다. 혹은 스트림 소스를 그대로 노출하고 컨슈머를 통해 도메인 별로 애플리케이션을 구성하는 `Data Mesh` 아키텍처로 접근하기도 합니다


## 1. 스트리밍 데이터 스테이징 토픽 생성
>  카파 아키텍처의 기본 스트리밍 파이프라인을 구성하여, 람다 아키텍처의 원본 테이블의 역할을 수행하는 스테이징 토픽을 생성합니다. 이러한 스테이징 소스를 생성 시에는 정보가 유실되지 않도록 유의해야 하며, 누구나 사용할 수 있는 데이터 소스를 가정하여 작성해야 하며, 생성된 토픽의 스키마 정보 (컬럼 이름, 데이터 타입 등)도 문서화 되어야 합니다
* stage pipeline : fluentd -> kafka -> spark-stream-stage -> kafka 

### 1.1 `fluentd` 통한 예제 데이터 생성
> fluentd dummy plugin 을 통해 `{id:number, time:timestamp}` 형식의 예제 데이터 생성

### 1.2 `kafka` 클러스터에 `events` 토픽에 저장된 데이터 확인
> `kafka-console-consumer`를 통해 저장된 토픽 데이터 확인

### 1.3 `kafka` 클러스터에 `names` 글로벌 토픽에 저장된 데이터 조인
> 카프카의 글로벌 테이블인 `names`에 0~5번 id 값에 매칭되는 값을 생성하여 저장

### 1.4  `spark streaming` 통해 생성된 토픽을 콘솔에 출력
> 스파크 스트리밍을 통해 `events` 토픽의 `id` 와 `names` 테이블의 `id, id_name` 컬럼을 조인하여 정상적인 조회를 확인합니다
> 이 토픽에는 원본 토픽을 보강(enrich)한 상태로 저장하되, 정보의 유실(집계 등)이 없는 상태로 저장하는 것이 일반적입니다

### 1.5 1차 가공된 데이터를 `events_stage` 토픽에 저장
> 데이터를 `events_stage`에 잘 저장되었는지 확인하고, 이 토픽이 스테이징된 데이터 원본으로 활용될 테이블(토픽)입니다 

### 1.6 스테이징 데이터를 `kafka-console`을 통해 확인합니다
> 카프카 서버에 접속 후, 콘솔에서 생성된 데이터가 정상적으로 조회 되는 지 확인합니다


## 2. 스트리밍 집계 파이프라인 생성
>  `event_stage` 토픽은 24시간 수신되는 스트리밍 로그 토픽이며, 이를 활용하여 1차 가공 데이터를 생성하고 이를 `MySQL` 테이블에 저장하는 예제를 실습합니다
* stream pipeline : kafka -> spark-stream-agg -> mysql -> phpmyadmin

### 2.1 별도의 스파크 애플리케이션을 하나 더 생성합니다
>  특정 도메인에 맞는 비즈니스 로직을 담는 애플리케이션을 개발할 것이며, 기존에 생성했던 `events` 토픽의 데이터는 계속 생성되고 있어야만 합니다

### 2.2 `user_name` 기준으로 빈도를 측정하여 콘솔에 출력합니다
>  스트리밍 테이블과 정적인 테이블 조인 후에 `name` 컬럼 기준으로 집계한 결과를 실시간으로 보여주는 `events` 라는 테이블에 적재합니다. 스트리밍 조인 및 집계 처리시에는 반드시 `Watermark` 를 적용하여 지연된 데이터에 대한 처리가 필요합니다. 즉, 무한정 집계를 위해 지연된 데이터를 기다릴 수 없기 때문에 제한된 시간 내에 도착한 데이터만 사용하고 그 이상 지연된 데이터를 사용하지 않겠다는 의미입니다

### 2.3 출력된 지표를 `MySQL` 테이블 `events_name` 테이블에 적재합니다
>  현재의 상태만 보여주기 보다는, 추후 복구나 트랜드를 통한 분석이 필요할 수 있으므로, 필요에 따라 일자 혹은 시간대 별로 지표를 추출하는 것이 좋으며, 편의상 일자 `yyyy-MM-dd` 와 `user_name` 까지 2개의 컬럼을 기준으로 집계합니다. 원하는 출력결과를 눈으로 확인 하고, 해당 데이터를 `events_name` 이라는 테이블로 싱크하게 됩니다. 이 때에 `MySQL`의 경우에는 `JDBC Sink` 를 통해 적재되어야 하므로 `foreachBatch` 라는 방식으로 통해서 적재되어야 합니다.

### 2.4 `events_name` 테이블에 실시간으로 잘 적재되는 지 확인합니다
> `phpmyadmin` 페이지를 통해서 실시간으로 조회하여 집계와 적재가 잘 되는지 확인합니다. 이 때에 생성된 테이블이 외부 서비스로 노출되는 실시간 애플리케이션의 하나가 될 수 있습니다.


## 3. 스트리밍 복구 파이프라인 생성
>  앞서 설명 드린 스트리밍 집계 파이프라인에 버그 혹은 문제가 뒤늦게 발견되어 복구가 필요한 경우에 배치 애플리케이션을 통해 갱신하는 방법도 있지만, 스트리밍 애플리케이션을 그대로 활용하는 방법도 있습니다. 지연된 로그가 발견되어 조금 더 긴 `Watermark`를 적용하거나, 데이터의 크기가 커지는 것을 감안하여 `CPU, Memory` 등의 리소스를 더 설정하는 등의 운영이 가능합니다

### 3.1 복구를 위한 스파크 애플리케이션을 생성합니다
>  기존에 `2.1`에서 생성했던 애플리케이션은 시나리오 상 하루가 지난 이후에 복구를 하는 것으로 잡았기 때문에 기존의 애플리케이션을 종료하고 해당 애플리케이션을 복사해서 활용하면 됩니다.

### 3.2 잘못된 차원과 지연로그를 고려하여 애플리케이션을 수정합니다
>  기존의 `name` 테이블의 추가된 데이터에 대한 차원이 추가되지 않아 널값으로 출력되는 부분을 수정하기 위해 `0~9`까지의 모든 값을 저장하여야 하며, `Watermark` 시간 등을 조정할 수 있습니다. 외에도 리소스에 대한 부분도 고려해야 하지만, 실습에서는 제외하였습니다.

### 3.3 해당 일자의 지표가 정상적으로 수정이 되는지 확인합니다
> `MySQL` 에 저장되는 이전 일자의 지표가 잘 수정되는 지 확인합니다
* backup pipeline : spark-stream-agg -> mysql


## 4. 스트리밍 파이프라인의 확장
### 4.1 유사한 방식으로 `mongodb` 에 집계 혹은 가공 데이터를 적재하는 예제를 실습합니다
> `mongo-express` 같은 도구를 통해 확인이 가능합니다
