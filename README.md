# Spring Redis
### 본 프로젝트는 아래의 소스코드를 기반으로 제작/개선되었습니다.<br>🔗 https://github.com/excelh11/spring_redis
* 학습 목표: 원본 소스 분석 후 성능 개선, 구조 리팩터링, RESTful 개선 등을 적용하기



<img width="643" height="887" alt="Image" src="https://github.com/user-attachments/assets/671180db-54eb-4112-9fbb-63a166f7d9ae" />


[플로우 확인하기](https://github.com/user-attachments/files/24050884/B._.pdf)



## 팀원 소개

### [Team Notion](https://www.notion.so/Redis_Cache-Project-2c4fc6992fc5804dbd26ea18e382ada5?source=copy_link)


| 김민식(팀장) | 한정연 | 오인준 | 박태오 | 박건영 |
|:------:|:------:|:------:|:------:|:------:|
| [GitHub](https://github.com/minsik321) | [GitHub](https://github.com/DOT-SOY) | [GitHub](https://github.com/01nJun) | [GitHub](https://github.com/teomichaelpark-glitch) | [GitHub](https://github.com/keonyeong4550/redisCache) |


## 주요 기능

1. 인기 검색어
2. 최근 검색어
3. 검색어 데이터 DB 저장 + Redis 랭킹 반영
![Image](https://github.com/user-attachments/assets/97dbc604-41aa-47bc-8688-51d2932618ed)
4. 테스트 데이터 생성 / 캐시 초기화 기능
![Image](https://github.com/user-attachments/assets/3856e110-7827-4b2f-933d-f8acf9dd0cc4)
![Image](https://github.com/user-attachments/assets/a9a4fa7b-5ad1-4937-9c16-0043139774f8)
5. Redis vs DB 성능 비교 기능
![Image](https://github.com/user-attachments/assets/a741d9b3-9635-49fd-b0a5-00c884e0ed90)
6. Redis 상태 확인 기능
![Image](https://github.com/user-attachments/assets/36f7c9ca-61d4-4f8f-9f82-ad7582c2c390)

## 프로젝트 개선 사항

아래는 이번 프로젝트에서 중점적으로 개선한 항목들입니다.

### # TestDataController의 GetMapping 제거 또는 RequestMapping 사용
#### 🔴 기존 코드
```
@PostMapping("/generate-data")
@GetMapping("/generate-data")
```

#### 🟢 개선 코드
```
@PostMapping("/generate-data")
// 또는
@RequestMapping("/generate-data")
```

#### 개선 이유

* 데이터 생성은 POST가 REST 규칙에 부합함
* GET 방식 제거로 리소스 혼동 방지
* RequestMapping 사용 시 중복 제거 및 유지보수성 향상

### # SearchService.getPopularKeywords()의 @Cacheable 제거
#### 🔴 기존 코드
```
@Cacheable(value = "search", key = "'popular_keywords'")
public List<String> getPopularKeywords(int limit) { ... }
```

#### 🟢 개선 코드
```
public List<String> getPopularKeywords(int limit) { ... }
```

#### 개선 이유

* 실시간 검색어는 초 단위로 변하므로 애플리케이션 캐시(@Cacheable) 적용 시 최신 데이터와 불일치
* Redis는 실시간 조회에 최적화(메모리 기반)
* 실시간 인기 검색어는 반드시 Redis 직접 조회가 맞는 구조

### # script 파일 search(btn) 수정
#### (addUserSearchKeyword / updatePopularKeywords 제거 후 loadKeywords 통합)


#### 🔴 기존 코드
```
addUserSearchKeyword(keyword);
await updatePopularKeywords();
```

#### 🟢 개선 코드
```
loadKeywords();
// addUserSearchKeyword(keyword);
// await updatePopularKeywords();
```

#### 개선 이유

* JS 로컬 저장 방식은 사용자별 분리/동기화가 불가능
* Redis 저장 시 장점:
* 사용자별 데이터 유지 가능
* 여러 기기에서 동일한 기록 조회 가능
* 검색어 분석, 추천 시스템 확장 가능
* 동일한 구조를 사용하여 화면 로직 단순화 및 유지보수성 향상

### # SearchService.processSearch()의 Redis 호출을 파이프라인으로 통합
#### 🔴 기존 코드 (Redis 명령 4회 왕복)

ZINCRBY-
LREM-
LPUSH-
LTRIM

```
updateRealTimeRanking(keyword);
updateRecentKeywords(keyword);
```

#### 🟢 개선 코드 (Pipeline으로 1회 왕복)
```
stringRedisTemplate.executePipelined(new SessionCallback<Object>() {
    @Override
    public Object execute(RedisOperations operations) {
        ZSetOperations<String, String> zOps = operations.opsForZSet();
        ListOperations<String, String> lOps = operations.opsForList();

        zOps.incrementScore(POPULAR_KEYWORDS_KEY, keyword, 1.0);
        lOps.remove(RECENT_KEYWORDS_KEY, 0, keyword);
        lOps.leftPush(RECENT_KEYWORDS_KEY, keyword);
        lOps.trim(RECENT_KEYWORDS_KEY, 0, 9);
        return null;
    }
});
```

#### 개선 이유

* 기존 방식은 Redis와의 왕복(RTT)이 4회 발생
* 파이프라인 적용 시 명령을 모아서 1번에 전송 → 성능 개선
* 관련 로직 통합으로 가독성과 유지보수 개선

### # updateRedisBulkOnly(): RedisConnection → 고수준 API로 교체

#### 🔴 기존 코드 (직렬화 직접 처리)
```
conn.zIncrBy(zkey, e.getValue(), ser.serialize(e.getKey()));
conn.lRem(lkey, 0, ser.serialize(kw));
```

#### 🟢 개선 코드 (opsForZSet / opsForList 사용)
```
zOps.incrementScore(POPULAR_KEYWORDS_KEY, keyword, delta);
lOps.remove(RECENT_KEYWORDS_KEY, 0, kw);
lOps.leftPush(RECENT_KEYWORDS_KEY, kw);
```

#### 개선 이유

* RedisConnection은 직렬화 직접 처리 → 코드 복잡도 증가
* Spring 제공 고수준 API는 직렬화 자동 처리 → 가독성 & 유지보수성 향상
* 성능은 동일하므로 실제 서비스에서는 고수준 API가 더 적합

### # clearAllCacheFast()에 @CacheEvict 추가 (데이터 일관성 확보)
#### 🔴 기존 코드
```
public void clearAllCacheFast() {
    stringRedisTemplate.delete(POPULAR_KEYWORDS_KEY);
    stringRedisTemplate.delete(RECENT_KEYWORDS_KEY);
}
```

#### 🟢 개선 코드
```
@CacheEvict(cacheNames = "search", allEntries = true)
public void clearAllCacheFast() {
    stringRedisTemplate.delete(POPULAR_KEYWORDS_KEY);
    stringRedisTemplate.delete(RECENT_KEYWORDS_KEY);
}
```

#### 개선 이유

* Redis 데이터 삭제와 Spring Cache 삭제는 별개
* Redis만 지우면 @Cacheable에 남은 오래된 데이터가 반환될 위험
* @CacheEvict 추가로 두 저장소의 데이터 일관성 보장


## 기술 스택

- Backend: Spring Boot, Spring Data Redis, Spring Cache
- Database: Redis, /MariaDB
- Frontend: HTML / JavaScript / CSS
