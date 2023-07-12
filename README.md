# Real-MySQL
Real MySQL 8.0 책을 통해 학습한 내용을 기록

<details markdown="1">
<summary> 

#### ***스토리지 엔진이란?***  </summary>  
dbms에 포함되는 컴포넌트로 crud를 담당해 물리적 저장장치에서 데이터를 읽어오는 역할을 담당한다.  

##

`innoDB 엔진`  
장점 : 트랜잭션이 지원되며, row-level-lock을 사용해 갱신 작업 속도가 빠르며 동시성 처리에 유리하다.  
단점 : 시스템 자원을 많이 사용한다.  

사용처 : 데이터 무결성이 필요하며 트랜잭션 처리가 중요한 작업에 사용.

`MyISAM 엔진`  
장점 : table 단위로 locking을 제공하며 read-only 위주의 간단한 작업에 유리하다.  
단점 : 트랜잭션이 지원되지 않으며 동시성 처리에 불리하다.  

사용처 : 트랜잭션이 필요없는 읽기 위주의 정적인 작업에 사용.  

`memory 엔진`  
heap 테이블이라 불리며 빠른 insert와 select 작업이 가능하다.  
트랜잭션이 지원되지 않으며, 모든 데이터를 RAM에 저장하기 때문에 휘발성이다.  
데이터 분석 시 중간 결과 저장용, JWT 사용시 RefreshToken 저장용으로 사용 가능하다.  

##

</details>


## ***트랜잭션과 잠금***

`트랜잭션이란?`  
트랜잭션은 작업의 완전성을 보장해주는 것이다.  
즉 논리적인 작업 셋을 모두 완벽하게 처리하거나, 처리하지 못할 경우에는 원 상태로 복구해서 작업의 일부만 적용되는 현상이 발생하지 않게 만들어 주는 기능이다.  


<details markdown="1">
<summary> 

#### ***1. 트랜잭션의 설계***  </summary>  

트랜잭션은 꼭 필요한 최소의 코드에만 적용하는것이 좋다.  

```
1) 처리시작

  => DB 커넥션 생성 
  => 트랜잭션 시작

2) 사용자의 로그인 여부 확인
3) 사용자의 글쓰기 내용의 오류 여부 확인
4) 첨부로 업로드된 파일 확인 및 저장
5) 사용자의 입력 내용을 DBMS에 저장
6) 첨부 파일 정보를 DBMS에 저장
7) 저장된 내용 또는 기타 정보를 DBMS에서 조회
8) 게시물 등록에 대한 알림 메일 발송
9) 알림 메일 발송 이력을 DBMS에 저장

  <= 트랜잭션 종료
  <= DB 커넥션 반납

10) 처리완료
```

위 처리 절차에는 문제가 있다. 

트랜잭션이 필요한 작업은 5)부터이다.   
db 커넥션은 개수가 제한적이어서 각 단위 프로그램이 커넥션을 소유하는 시간이 길어질수록 여유 커넥션이 줄어들어 커넥션을 대기하는 시간이 길어지게된다.  
8)은 트랜잭션에 포함시키면 안된다. 메일 전송이나 FTP 파일 전송 작업 또는 네트워크를 통해 원격 서버와 통신하는 작업은 트랜잭션에서 제거하는것이 좋다.  
프로그램이 실행되는 동안 메일 서버와 통신할 수 없는 상황이 발생한다면 웹 서버뿐 아니라 DBMS 서버까지 위험해지는 상황이 발생한다.  

아래와 같이 불필요한 작업을 트랜잭션 내에서 제거하고 연관된 작업을 묶어 트랜잭션을 적용해 볼 수 있다.  

```
1) 처리시작
2) 사용자의 로그인 여부 확인
3) 사용자의 글쓰기 내용의 오류 여부 확인
4) 첨부로 업로드된 파일 확인 및 저장

  => DB 커넥션 생성 
  => 트랜잭션 시작

5) 사용자의 입력 내용을 DBMS에 저장
6) 첨부 파일 정보를 DBMS에 저장
  
  <= 트랜잭션 종료
  
7) 저장된 내용 또는 기타 정보를 DBMS에서 조회
8) 게시물 등록에 대한 알림 메일 발송
  => 트랜잭션 시작
9) 알림 메일 발송 이력을 DBMS에 저장

  <= 트랜잭션 종료
  <= DB 커넥션 반납

10) 처리완료
```
</details>




<details markdown="1">
<summary> 

#### ***2. MySQL 엔진의 잠금***  </summary>  

MySQL에서의 잠금은 크게 `스토리지 엔진 레벨`과 `MySQL 엔진 레벨`로 나눌 수 있다.  
스토리지 엔진을 제외한 나머지 부분이다.  

#### 영향도  
MySQL엔진 레벨에서의 잠금은 `모든 스토리지 엔진에 영향을 미친다.`  
스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 `상호 영향을 미치지는 않는다.`  

`MySQL 엔진`에서는,,,  
테이블 데이터 동기화를 위한 `테이블 락`  
테이블의 구조를 잠그는 `메타 데이터 락`  
사용자의 필요에 맞게 사용할 수 있는 `네임드 락` 기능을 제공한다.  



#### 1) 글로벌 락  
```
FLUSH TABLES WITH READ LOCK 명령으로 획득할 수 있다.
```
MySQL에서 획득할 수 있는 가장 범위가 큰 락이 글로벌 락이다.  
글로벌 락 명령어는 테이블에 실행 중인 모든 종류의 쿼리가 완료되야 한다.  
한 세션에서 글로벌 락을 획득하면 다른 세션에서 SELECT를 제외한 대부분의 DDL 문장이나 DML 문장을 실행하는 경우 글로벌 락이 해제될 때까지 해당 문장이 대기 상태로 남는다.  
이러한 이유 때문에, 글로벌 락은 MySQL 서버의 모든 테이블에 큰 영향을 미치기 때문에 웹 서비스용으로 사용되는 MySQL 서버에서는 가급적 사용하지 않는 것이 좋다.  

하지만, MySQL 서버가 업그레이드되면서 MyISAM이나 MEMORY 스토리지 엔진 보다는 InnoDB 스토리지 엔진의 사용이 일반화되었다.  
InnoDB 엔진은 트랜잭션을 지원하기 떄문에 데이터 일관성을 위해 모든 변경 작업을 멈출 필요는 없다.  
또, MySQL 8.0부터 InnoDB가 기본 스토리지 엔진으로 채택되면서 조금 더 가벼운 글로벌 락의 필요성이 생겼다.  
그래서 Xtrabackup이나 Enterprise Backup과 같은 백업 툴들의 안정적인 실행을 위해 백업 락이 도입됐다.  

```
LOCK INSTANCE FOR BACKUP;
==> 백업 실행
UNLOCK INSTANCE;
```

특정 세션에서 백업 락을 획득하면 모든 세션에서 아래 작업을 수행할 수 없다.  
- 데이터베이스 및 테이블 등 모든 객체 생성 및 변경 삭제  
- REPAIR TABLE과 OPTIMIZE TABLE 명령  
- 사용자 관리 및 비밀번호 변경  
하지만, 백업 락은 일반적인 테이블의 데이터 변경은 허용된다.  


#### 2) 테이블 락  
`개별 테이블 단위로 설정되는 잠금이며, 명시적 또는 묵시적으로 락을 획득할 수 있다.`  

##### 명시적 락
명시적 락은 온라인 작업에 상당한 영향을 미치기 때문에 사용할 필요가 거의 없다.  
```
획득
LOCK TABLES table_name [ READ | WRITE ] 

해제
UNLOCK TABLES
```  

##### 묵시적 락
MyISAM, MEMORY 테이블에 사용하며 쿼리가 실행되는 동안 자동으로 획득됐다가, 완료된 후 자동 해제된다.  
InnoDB의 경우 스토리지 엔진 차원에서 레코드 기반의 잠금을 제공하기 때문에 단순 데이터 변경 쿼리로 인해 묵시적인 락이 설정되지는 않는다.  
InnoDB 테이블에도 테이블 락이 설정되지만 데이터 변경 쿼리에서는 무시되고 스키마를 변경하는 DDL 에서만 적용된다.  


#### 3) 네임드 락  
네임드락은 GET_LOCK() 함수를 통해 획득하며 사용자가 지정한 임의의 문자열에 대해 잠금을 설정할 수 있다.  
배치 프로그램처럼 한꺼번에 많은 레코드를 변경하는 경우, 복잡한 요건으로 레코드를 변경하는 트랜잭션에 유용하며  
같은 데이터를 참조하는 프로그램끼리 분류해서 네임드 락을 걸고 쿼리를 실행하면 된다.  



#### 4) 메타 데이터 락  
데이터베이스 객체( 테이블, 뷰 )의 이름이나 구조를 변경하는 경우에 묵시적으로 획득하는 잠금이다.


#### 5) InnoDB 스토리지 엔진 잠금  
MySQL에서 제공하는 잠금과는 별개로 스토리지 엔진 내부에서 `레코드 기반의 잠금 방식`을 탑재하고 있음.  

`레코드 락`  
레코드 자체만을 잠그는 것을 레코드 락이라고 한다.  
InnoDB 스토리지 엔진은 레코드 자체가 아니라 `인덱스의 레코드`를 잠근다.  
인덱스가 없더라도 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정한다.  

```
employees 테이블에 약 30만건의 데이터가 있고, 
만약 first_name이 Georgi인 사원은 253명, 앞의 조건과 last_name이 Klassen인 사원은 단 1명만 존재한다고 가정해본다.  
멤버로 담긴 ix_firstname이라는 인덱스가 준비되어있다고 가정해본다.
KEY ix_firstname (first_name)

UPDATE employees SET hire_date=NOW() WHERE first_name='Georgi' AND last_name='Klassen'
위의 쿼리를 실행하게 된다면 first_name에 인덱스가 걸려 있기 때문에 253건의 레코드가 모두 잠길것이다.  
만약 인덱스가 없다면 30만건의 레코드가 전부 잠금에 걸릴 것이다.
때문에 InnoDB에서 인덱스 설계가 중요한 것이다.
```


- 갭 락  
레코드 자체가 아니라 레코드와 바로 인접한 레코드 사이의 간격만을 잠그는 것을 의미한다.  
레코드와 레코드 사이의 간격에 새로운 레코드가 INSERT 되는 것을 제어한다.  

- 넥스트 키 락  
레코드 락과 갭 락을 합쳐 놓은 형태의 잠금을 넥스트 키 락 이라고 한다. (레코드의 앞 뒤도 잠김)  


#### 6) 자동 증가 락
AUTO_INCREMENT 컬럼이 사용된 테이블에 동시에 여러 레코드가 INSERT 되는 경우, 저장되는 각 레코드가 중복되지 않게 InnoDB 스토리지 엔진 내부에서 AUTO_INCREMENT 락 이라고 하는 테이블 수준의 잠금을 사용한다.  
INSERT, REPLACE 쿼리와 같이 새로운 레코드를 저장하는 경우에만 사용되며, UPDATE나 DELETE의 경우에는 걸리지 않는다.  
트랜잭션과 상관없이 AUTO_INCREMENT 값을 가져오는 순간만 락이 걸렸다가 즉시 해제된다.  
MySQL 5.1 이상부터는 innodb_autoinc_lock_mode라는 시스템 변수를 통해 작동방식 변경이 가능하다.  
</details>


##



<details markdown="1">
<summary> 

#### ***INDEX***  </summary>  
  

### Explain (실행계획)  
MYSQL 옵티마이저가 수립한 실행 계획의 큰 흐름을 보여준다.  


##

###### select_type 컬럼  
각 단위 SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 컬럼이다.  

`SIMPLE`  
UNION이나 서브쿼리를 사용하지 않는 단순한 SELECDT 쿼리의 경우 SIMPLE이 표시됨.  
쿼리가 복잡하더라도 SIMPLE인 단위 쿼리는 하나만 존재한다.  
일반적으로 제일 바깥 SELECT 쿼리의 select_type이 SIMPLE로 표시됨.  

`PRIMARY`  
UNION이나 서브쿼리를 가지는 SELECT 쿼리의 가장 바깥쪽에 있는 단위 쿼리는 PRIMARY로 표시된다.  
PRIMARY인 단위 쿼리는 하나만 존재한다.  

`UNION`  
UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리의 select_type은 UNION으로 표시됨.  
첫번째 단위는 UNION되는 쿼리 결과들을 모아서 저장하는 임시 테이블(DERIVED)가 표시됨.  

`DEPENDENT UNION`  
UNION이나 UNION ALL로 집합을 결합하는 쿼리에서 표시된다.  
DEPENDENT는 UNION이나 UNION ALL로 결합된 단위 쿼리가 외부 쿼리에 의해 영향을 받는 것을 의미한다.  

ex)  
EXPLAIN  
SELECT *  
FROM employees e1 WHERE e1.emp_no IN (  
  SELECT e2.emp)no FROM employees e2 WHERE e2.first_name='Matt'    
  UNION  
  SELECT e3.emp_no FROM employees e3 WHERE e3.last_name='Matt'  
);  

`UNION RESULT`  
UNION RESULT는 UNION 결과를 담아두는 테이블을 의미한다.  
select_type이 UNION RESULT인 경우 EXPLAIN table 컬럼에는 <union1, 2>와 같은 값이 출력된다.  
이는 explain의 id 1, 2의 조회 결과를 UNION 했다는 것을 의미한다.  

##

###### partitions 컬럼  

,,,  
PARTITION BY RANGE COLUMNS(hire_date)  
(PARTITION p1986_1990 VALUES LESS THAN ('1990-01-01'),  
PARTITION p1991_1995 VALUES LESS THAN ('1996-01-01'),  
PARTITION p1996_2000 VALUES LESS THAN ('2000-01-01'),  
PARTITION p2001_2005 VALUES LESS THAN ('2006-01-01'));  

EXPLAIN  
SELECT *  
FROM employees  
WHERE hire_date BETWEEN '1995-11-15' AND '2000-01-15';  
  
위와 같은 경우, explain의 partitions 컬럼에는 p1996_2000, p2001_2005라고 표현되며 type에는 ALL이라고 표기된다.  
왜 풀스캔(ALL)으로 표현될까?  
MYSQL에서 지원하는 파티션은 물리적으로 개별 테이블처럼 별도의 저장공간을 가지기 때문이다.  
이 쿼리의 경우 모든 파티션이 아니라 p1996_2000, p2001_2005 파티션만 풀 스캔한다는 의미이다.  





##

###### rows 컬럼  
옵티마이저 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수를 보여준다.  
rows 컬럼에 표시되는 값은 반환하는 레코드의 예측치가 아니라, 쿼리를 처리하기 위해 얼마나 많은 레코드를 읽고 체크해야 하는지를 의미한다.  

##


###### Extra 컬럼  
쿼리의 실행 계획에서 성능에 관련된 내용이 표시된다.  
쿼리 실행에 있어 OO을 더 실행하겠다. 라는 내용이기에 NULL이 제일 좋다.  




`Using filesort`  
ORDER BY 처리가 인덱스를 사용하지 못할 때만 실행 계획의 Extra 컬럼에 Using filesort 코멘트가 표시된다.  
이는 조회된 레코드를 정렬용 메모리 버퍼에 복사해 퀵 소트 또는 힙 소트 알고리즘을 이용해 정렬을 수행하게 된다는 의미이다.  
MYSQL 옵티마이저는 레코드를 읽어서 **소트 버퍼에 복사하고, 정렬해서 그 결과를 클라이언트에 보낸다.  
이러한 쿼리는 많은 부하를 일으키므로, 쿼리를 튜닝하거나 인덱스를 생성하는 것이 좋다.  

** 소트 버퍼  
정렬을 수행하기 위해 별도의 메모리 공간을 할당 받아서 사용하는데, 이 메모리 공간을 소트 버퍼라고 한다.  


##
`Using index(커버링 인덱스)`  
데이터 파일을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있는 경우.  
인덱스를 통해 쿼리를 처리하는 경우 가장 큰 부하를 차지하는 부분은, 인덱스 검색에서 일치하는 키 값들의 레코드를 읽기 위해 데이터 파일을 검색하는 작업이다.  

ex)   
select first_name, birth_date  
from employees  
where first_name BETWEEN 'babette' AND 'Gad';  

where절에 일치하는 레코드가 5만건이라 할때,  
이 쿼리가 인덱스(Ix_firstname)을 사용한다면 일치하는 레코드 5만여 건을 검색하고, 각 레코드의 birth_date 컬럼의 값을 읽기 위해 각 레코드가 저장된 데이터 페이지를 5만여 번 읽어야 한다.  
이러한 경우 MYSQL 옵티마이저가 인덱스 보다 풀스캔으로 처리하는 것이 더 효율적이라고 판단할 수 있다.  

ex)

select first_name
from employees  
where first_name BETWEEN 'babette' AND 'Gad';  

이러한 경우 해당 테이블의 first_name 컬럼만 있으면 쿼리를 수행할 수 있다.  
이와 같이 인덱스만으로 처리되는 것을 "커버링 인덱스"라고 한다.  


InnoDB의 모든 테이블은 클러스터링 인덱스로 구성되어있다.  
인덱스의 "레코드 주소" 값에 테이블의 프라이머리 키가 저장되어 있다.  

ex)  
select emp_no, first_name
from employees  
where first_name BETWEEN 'babette' AND 'Gad';  

위의 예제들과 다르게 테이블의 프라이머리 키는 이미 인덱스에 포함돼 있어 데이터 파일을 읽지 않아도 되며, 인덱스를 레인지 스캔으로 처리한다.  


##
`Using temporary`    
MYSQL 서버에서 쿼리를 처리하는 동안 중간 결과를 담아 두기 위해 임시 테이블을 사용한다.  
임시 테이블은 메모리상에 생성될 수도, 디스크상에 생성될 수도 있다.  

대표적으로 임시 테이블을 생성하는 쿼리.  
```
1. FROM 절에 사용된 서브쿼리는 무조건 임시 테이블을 생성함. 이 테이블을 파생 테이블 이라고 부른다.  
2. COUNT(DISTINCT column1)을 포함하는 쿼리도 인덱스를 사용할 수 없는 경우에는 임시 테이블이 만들어진다.  
3. UNION이나 UNION DISTINCT가 사용된 쿼리도 항상 임시 테이블을 사용해 결과를 병합한다. (MYSQL 8.0 부터는 UNION ALL이 사용된 경우에는 임시 테이블을 사용하지 않도록 개선됨)  
4. 인덱스를 사용하지 못하는 정렬 작업 또한 임시 버퍼 공간을 사용하는데, 정렬해야 할 레코드가 많아지면 결국 디스크를 사용한다.
정렬에 사용되는 소트 버퍼 또한 임시 테이블과 같다.  
```


##

`Using Where`  
조인, 필터링, 집한처리,,, 등을 처리하는 MYSQL 엔진 레이어에서 별도의 가공을 통해 필터링(여과) 작업을 처리한 경우에 Extra 컬럼에 Using where 코멘트가 표시된다.  


##

`Zero limit`  
데이터 값이 아닌 쿼리 결과값의 메타데이터만 필요한 경우 쿼리의 마지막에 LIMIT 0을 사용하면 Zero limit 메시지가 출력된다.  


</details>












