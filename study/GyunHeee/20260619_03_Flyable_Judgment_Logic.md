# 환경 모니터링과 드론 운용 가능 여부(is_flyable) 판단 연결

## 관련 테이블 구조

```sql
-- 기상 데이터
CREATE TABLE weather_data (
  id              INT PRIMARY KEY AUTO_INCREMENT,
  vertiport_id    INT NOT NULL,
  wind_speed_ms   FLOAT,        -- 풍속 (m/s)
  wind_gust_ms    FLOAT,        -- 돌풍 (m/s)
  precipitation   FLOAT,        -- 강수량 (mm/h)
  visibility_m    INT,          -- 가시거리 (m)
  temperature_c   FLOAT,
  recorded_at     DATETIME NOT NULL
);

-- RF 환경 데이터
CREATE TABLE rf_environment (
  id              INT PRIMARY KEY AUTO_INCREMENT,
  vertiport_id    INT NOT NULL,
  signal_strength INT,          -- dBm
  interference_level ENUM('none', 'low', 'medium', 'high'),
  gps_quality     INT,          -- 위성 수신 품질 (0~100)
  recorded_at     DATETIME NOT NULL
);

-- 버티포트 운용 상태
CREATE TABLE vertiport_status (
  id              INT PRIMARY KEY AUTO_INCREMENT,
  vertiport_id    INT NOT NULL,
  is_flyable      BOOLEAN NOT NULL DEFAULT FALSE,
  reason          VARCHAR(500),    -- 불가 사유
  evaluated_at    DATETIME NOT NULL
);
```

---

## 판단 로직: is_flyable 계산

```typescript
// flyable-evaluator.service.ts
@Injectable()
export class FlyableEvaluatorService {

  // 판단 기준값 (국토교통부 드론 운용 기준 참고)
  private readonly LIMITS = {
    maxWindSpeed: 10,     // 풍속 10m/s 초과 시 불가
    maxWindGust: 15,      // 돌풍 15m/s 초과 시 불가
    maxPrecipitation: 1,  // 강수량 1mm/h 초과 시 불가
    minVisibility: 500,   // 가시거리 500m 미만 시 불가
    minGpsQuality: 60,    // GPS 품질 60 미만 시 불가
    maxInterference: 'medium',  // RF 간섭 medium 이상 시 불가
  };

  evaluate(weather: WeatherData, rf: RfEnvironment): FlyableResult {
    const violations: string[] = [];

    // 기상 조건 검사
    if (weather.windSpeedMs > this.LIMITS.maxWindSpeed) {
      violations.push(`풍속 초과: ${weather.windSpeedMs}m/s (기준: ${this.LIMITS.maxWindSpeed}m/s)`);
    }
    if (weather.windGustMs > this.LIMITS.maxWindGust) {
      violations.push(`돌풍 초과: ${weather.windGustMs}m/s`);
    }
    if (weather.precipitation > this.LIMITS.maxPrecipitation) {
      violations.push(`강수 감지: ${weather.precipitation}mm/h`);
    }
    if (weather.visibilityM < this.LIMITS.minVisibility) {
      violations.push(`가시거리 부족: ${weather.visibilityM}m`);
    }

    // RF 환경 검사
    if (rf.gpsQuality < this.LIMITS.minGpsQuality) {
      violations.push(`GPS 품질 불량: ${rf.gpsQuality}`);
    }
    if (this.interferenceLevel(rf.interferenceLevel) >= this.interferenceLevel('high')) {
      violations.push(`RF 간섭 심각: ${rf.interferenceLevel}`);
    }

    return {
      isFlyable: violations.length === 0,
      reason: violations.join('; ') || null,
    };
  }

  private interferenceLevel(level: string): number {
    return { none: 0, low: 1, medium: 2, high: 3 }[level] ?? 0;
  }
}
```

---

## 판단 → DB 저장 → 이벤트 발생 흐름

```typescript
// vertiport-monitor.service.ts
@Injectable()
export class VertiportMonitorService {

  // 5분마다 판단 갱신
  @Interval(5 * 60 * 1000)
  async evaluateAllVertiports(): Promise<void> {
    const vertiports = await this.vertiportRepo.find();

    for (const vertiport of vertiports) {
      const latestWeather = await this.getLatestWeather(vertiport.id);
      const latestRf = await this.getLatestRf(vertiport.id);

      if (!latestWeather || !latestRf) continue;

      const result = this.evaluator.evaluate(latestWeather, latestRf);

      // 이전 상태와 비교
      const previous = await this.statusRepo.findOne({
        where: { vertiportId: vertiport.id },
        order: { evaluatedAt: 'DESC' },
      });

      // 상태 변경 시에만 이벤트 발생
      if (!previous || previous.isFlyable !== result.isFlyable) {
        await this.statusRepo.save({
          vertiportId: vertiport.id,
          isFlyable: result.isFlyable,
          reason: result.reason,
          evaluatedAt: new Date(),
        });

        // 운용 불가로 전환 시 즉시 알람
        if (!result.isFlyable) {
          this.alertService.pushAlert({
            type: 'FLYABLE_STATUS_CHANGED',
            vertiportId: vertiport.id,
            isFlyable: false,
            reason: result.reason,
          });
        }
      }
    }
  }
}
```

---

## 전체 데이터 흐름

```
기상 센서 / 외부 기상 API
    ↓ 5분마다 수집
weather_data 테이블 INSERT
    ↓
RF 모니터 장비
    ↓
rf_environment 테이블 INSERT
    ↓
FlyableEvaluatorService.evaluate()
    ↓
vertiport_status 테이블 UPDATE
    ↓
is_flyable 변경 시 → AlertService.pushAlert()
    ↓
SSE로 React에 즉시 전달
    ↓
관리자 화면에 운용 가능 여부 표시
```

---

## React에서의 표현

```
[버티포트 A]  🟢 운용 가능
[버티포트 B]  🔴 운용 불가 — 풍속 초과: 12.3m/s, 강수 감지: 2.1mm/h
```

`is_flyable = false`이면 드론 이륙 명령 버튼을 비활성화하거나 경고를 표시한다.

---

## 주의: 판단은 보조 도구

`is_flyable` 판단은 수집된 센서값 기반의 **자동 보조 판단**이다.
최종 운용 결정은 운용자(사람)가 내린다는 전제를 시스템 설계에 명시해두어야 한다.
센서 오류나 예외 기상 상황에서 자동 판단만 믿으면 위험하다.
