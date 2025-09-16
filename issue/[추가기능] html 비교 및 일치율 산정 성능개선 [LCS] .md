### 요구사항

- Asis Tobe 시스템 검증을 위해 3가지형태의 response 비교 및 일치율 산정을 통한 결과처리
- 이를 확인 할 수 있는 직관적인 UI
- response 유형 3가지
  ```
  1. html (전체 / 부분)
  2. json
  3. String (pipe 구분자)
  ```
  
### 구현 방안

1. 태그단위 전처리 이후 LCS 알고리즘 기반 비교 ( reference : beyound compare)
   - LCS(Longest Common Subsequence) DP로 구현
   - 줄단위비교
   - 비교제외 영역
   - 같은 로직으로 직관적인 UI 구현
2. JsonNode로 파싱, 재귀로 노드 순회 비교
3. | 기준 문자열 분할 및 비교


### 성능개선

- 초기 문제점
  ```
  1. OOM 발생
  2. 비효율적인 전처리
  3. 응답 지연
  ```

- 원인
  ```
  분할 비교가 어려워 수천줄의 HTML을 메모리에 로드
  비교 제외 영역에 관한 처리시 index 변경 및 불필요한 탐색 (제외 영역이 추가될수록 Loof 횟수 증가)
  자체 구현 LCS 알고리즘 성능 한계
  ```

- 개선
  ```
  1.코드개선
  비교제외 영역 전처리 시 Set<Integer>로 index를 저장, 한번의 loof로 제외 영역 저장 O(1)
  Stream API로 일괄 필터, 반복탐색 제거
  불필요한 중간객체 생성 최소화

  2. java-diff-utils 라이브러리 도입
    Patch객체로 변경점을 유형별로 체계적으로 제공하여 정확한 라인별 비교를 ui로 구현하기 좋은 라이브러리라고 판단
    git의 4가지 diff 알고리즘 중 하나임 
   
  ```



