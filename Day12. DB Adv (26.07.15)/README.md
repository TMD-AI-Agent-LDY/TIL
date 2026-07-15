# Day12. DB Adv (26.07.15)

#### N : 1

- Many-to-One
  ![001.png](images/001.png)
- Foreign Key
  - RDB에서 다른 테이블의 행을 식별할 수 있는 키
  - 참조하는 테이블의 외래 키 값은 참조되는 테이블의 기본 키 (Primary Key)를 가리킴
    ![002.png](images/002.png)
    ![003.png](images/003.png)
    ![004.png](images/004.png)
- JOIN
  ![005.png](images/005.png)
  ![006.png](images/006.png)
  ![007.png](images/007.png)
  ![008.png](images/008.png)

#### M : N

- Many-to-Many
  ![009.png](images/009.png)
  - Many-to-Many 관계에선 테이블끼리 직접 연결하지 않는다.
  - 연결을 위한 테이블 (중간 테이블) 생성
    ![010.png](images/010.png)
    ![011.png](images/011.png)
    ![012.png](images/012.png)
    ![013.png](images/013.png)
    ![014.png](images/014.png)
  - Join
    ![015.png](images/015.png)
  - 1001번 주문에 어떤 상품이 들어갔는지 보고 싶다면 어떻게 작성할까?
    ![016.png](images/016.png)

#### 참고 자료

- Load & Dump
  - Load
    - 외부 데이터를 DB 안으로 넣는 것
      ![017.png](images/017.png)
  - Dump
    - DB 안의 데이터를 밖으로 빼는 것
      ![018.png](images/018.png)
  - python to sql
    - sqlite3
      ![019.png](images/019.png)
    - commit()
      ![020.png](images/020.png)
    - fetchall()
      ![021.png](images/021.png)
- 1 : 1
  ![022.png](images/022.png)
  ![023.png](images/023.png)
  - 테이블 분리 목적
    - 자주 쓰는 정보와 가끔 쓰는 정보를 분리하고 싶을 때
    - 민감한 정보를 따로 관리하고 싶을 때
    - 테이블이 너무 커지는 것을 피하고 싶을 때
    - 특정 정보만 권한을 다르게 관리하고 싶을 때
    - 선택적으로 존재하는 상세정보를 분리하고 싶을 때
- ERD
  - 데이터베이스의 구조를 시각적으로 표현하는 도구 개체 간의 관계를 그림으로 표현하는 다이어그램
  - ERD 구성 요소
    - 개체 (Entity)
      - 객체 및 개념 (명사)
      - User, Product, Article, Chat
    - 속성 (Attribute)
      - 개체의 특성 및 성질
      - User : 이름, 아이디, 관리자 여부
    - 관계 (Relationship)
      - 개체 간의 관계
      - User - Order - Product
  - 수적관계
    ![024.png](images/024.png)
    ![025.png](images/025.png)
- Supabase
  - PostgreSQL 기반 백엔드 플랫폼
  - Database
  - 인증 (Auth)
  - 파일 저장

#### 미니 PJT

- Code 폴더 참고.
