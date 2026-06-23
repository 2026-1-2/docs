# 온프레미스 서버 디스크(HDD)가 가득 찼을 때의 대응 정책

## 문제 상황

HDD가 가득 차면:
- MediaMTX가 새 세그먼트 파일을 쓰지 못해 녹화 중단
- MySQL이 binlog나 임시 파일을 못 써서 쿼리 실패
- NestJS 로그 파일 쓰기 실패 → 서비스 불안정

---

## 1. 예방: storage_policies 테이블 기반 자동 삭제

```sql
CREATE TABLE storage_policies (
  id INT PRIMARY KEY AUTO_INCREMENT,
  camera_id INT NOT NULL,
  retention_days INT NOT NULL DEFAULT 30,  -- 보관 기간
  max_size_gb FLOAT,                        -- 최대 용량 (선택)
  FOREIGN KEY (camera_id) REFERENCES cameras(id)
);
```

NestJS cron job이 매일 만료된 파일을 삭제한다 (→ topic 11 참고).

---

## 2. 감지: 디스크 사용량 모니터링

### NestJS에서 주기적으로 확인

```typescript
import { execSync } from 'child_process';

@Cron('0 * * * *')  // 매시간
async checkDiskUsage() {
  const output = execSync("df /mnt/hdd1 | tail -1 | awk '{print $5}'")
    .toString()
    .trim();  // "85%" 형태로 반환

  const usagePercent = parseInt(output);

  if (usagePercent >= 90) {
    this.logger.warn(`HDD 사용량 경고: ${usagePercent}%`);
    await this.alertService.pushAlert({ type: 'DISK_WARNING', usage: usagePercent });
  }

  if (usagePercent >= 95) {
    // 긴급: 가장 오래된 파일부터 강제 삭제
    await this.emergencyCleanup();
  }
}
```

---

## 3. 대응 정책 단계별 정의

| 사용량 | 대응 |
|--------|------|
| 80% | 관리자에게 SSE/알람 경고 |
| 90% | 보관 기간 초과 파일 즉시 삭제 실행 |
| 95% | 가장 오래된 녹화부터 강제 삭제, 신규 녹화 일시 중단 고려 |
| 99% | 전체 신규 녹화 중단, 긴급 수동 개입 필요 |

---

## 4. 긴급 상황 수동 대응

```bash
# 현재 HDD 사용량 확인
df -h /mnt/hdd1

# 가장 용량 큰 파일 확인
du -sh /mnt/hdd1/recordings/* | sort -rh | head -20

# 날짜 기준으로 오래된 파일 삭제 (30일 초과)
find /mnt/hdd1/recordings -name "*.mp4" -mtime +30 -delete

# 삭제 후 용량 확인
df -h /mnt/hdd1
```

---

## 5. 구조적 해결: 카메라별 할당량

```sql
-- camera_storage_quotas 테이블
camera_id  quota_gb   current_usage_gb
    1         50            42.3
    2         30            28.1
```

카메라별로 최대 할당량을 정하고, 초과 시 해당 카메라의 가장 오래된 녹화부터 삭제한다.
한 카메라가 전체 HDD를 독점하는 것을 방지한다.

---

## 핵심 원칙

디스크가 가득 차는 것은 **예측 가능한 장애**다.
사후 대응보다 사전 임계값 알람 + 자동 삭제 정책으로 예방하는 것이 핵심이다.
