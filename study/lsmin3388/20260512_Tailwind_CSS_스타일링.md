# Tailwind CSS 유틸리티 스타일링

## 1. 개요

버티포트 CCTV 감시 시스템의 프론트엔드는 다크 모드 기반의 대시보드 UI를 제공한다.
Tailwind CSS의 유틸리티 퍼스트 접근 방식을 활용하면 별도의 CSS 파일 없이
HTML 클래스만으로 일관된 디자인 시스템을 구축할 수 있다.
본 문서에서는 Tailwind CSS의 핵심 개념과 프로젝트 UI 컴포넌트 스타일링 전략을 분석한다.

---

## 2. CSS-in-JS vs Tailwind CSS 비교

| 항목 | CSS-in-JS (styled-components) | Tailwind CSS |
|------|-------------------------------|-------------|
| 스타일 정의 | JS 파일 내 템플릿 리터럴 | HTML 클래스 유틸리티 |
| 번들 크기 | 런타임 JS 포함 (~12KB) | 빌드 시 사용 클래스만 추출 (~10KB) |
| 런타임 오버헤드 | 있음 (CSS 동적 생성) | 없음 (정적 CSS) |
| 디자인 토큰 | theme 객체로 관리 | tailwind.config.js로 관리 |
| 러닝 커브 | CSS 지식 + API 학습 | 유틸리티 클래스명 숙지 |
| 조건부 스타일 | props 기반 (`${props => ...}`) | 클래스 조합 (`cn()` 함수) |
| SSR 호환 | 설정 필요 | 완벽 호환 |
| 컴포넌트 결합 | 스타일과 컴포넌트 강결합 | 느슨한 결합 |

---

## 3. 주요 유틸리티 클래스

### 3-1. 레이아웃

| 클래스 | CSS 속성 | 설명 |
|--------|----------|------|
| `flex` | `display: flex` | 플렉스 컨테이너 |
| `grid` | `display: grid` | 그리드 컨테이너 |
| `grid-cols-2` | `grid-template-columns: repeat(2, 1fr)` | 2열 그리드 |
| `gap-4` | `gap: 1rem` | 간격 16px |
| `w-full` | `width: 100%` | 전체 너비 |
| `h-screen` | `height: 100vh` | 뷰포트 높이 |
| `absolute` | `position: absolute` | 절대 위치 |
| `z-10` | `z-index: 10` | z축 순서 |
| `overflow-hidden` | `overflow: hidden` | 넘침 숨김 |

### 3-2. 타이포그래피

| 클래스 | CSS 속성 | 설명 |
|--------|----------|------|
| `text-sm` | `font-size: 0.875rem` | 작은 글자 |
| `text-lg` | `font-size: 1.125rem` | 큰 글자 |
| `font-bold` | `font-weight: 700` | 굵은 글자 |
| `text-gray-400` | `color: rgb(156 163 175)` | 회색 텍스트 |
| `text-center` | `text-align: center` | 가운데 정렬 |
| `truncate` | `overflow: hidden; text-overflow: ellipsis; white-space: nowrap` | 말줄임 |
| `leading-tight` | `line-height: 1.25` | 좁은 줄간격 |

### 3-3. 색상 및 배경

```tsx
// 배경색
<div className="bg-gray-900" />      // 어두운 배경
<div className="bg-blue-500/20" />    // 투명도 20% 파란 배경

// 텍스트 색상
<span className="text-white" />       // 흰색 텍스트
<span className="text-red-500" />     // 빨간 텍스트 (경고)
<span className="text-emerald-400" /> // 녹색 텍스트 (정상)

// 테두리
<div className="border border-gray-700 rounded-lg" />
<div className="ring-2 ring-blue-500" />
```

### 3-4. 반응형 디자인

Tailwind는 모바일 퍼스트 breakpoint 시스템을 사용한다.

| 접두사 | 최소 너비 | 설명 |
|--------|-----------|------|
| (없음) | 0px | 모바일 기본 |
| `sm:` | 640px | 소형 태블릿 |
| `md:` | 768px | 태블릿 |
| `lg:` | 1024px | 노트북 |
| `xl:` | 1280px | 데스크탑 |
| `2xl:` | 1536px | 대형 모니터 |

```tsx
// 반응형 그리드: 모바일 1열 → 태블릿 2열 → 데스크탑 4열
<div className="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-4 gap-4">
  {cameras.map((cam) => (
    <CameraCard key={cam.id} camera={cam} />
  ))}
</div>
```

---

## 4. 다크 모드 설정

### 4-1. tailwind.config.js 설정

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  darkMode: 'class',  // 'class' 전략: <html class="dark">로 전환
  theme: {
    extend: {
      colors: {
        // 프로젝트 커스텀 색상
        'mmp-bg': '#0f1117',
        'mmp-card': '#1a1d27',
        'mmp-border': '#2a2d37',
        'mmp-accent': '#3b82f6',
      },
    },
  },
  plugins: [],
};
```

### 4-2. 다크 모드 적용

```tsx
// 다크 모드 클래스 사용
<div className="bg-white dark:bg-gray-900 text-black dark:text-white">
  <h1 className="text-gray-900 dark:text-gray-100">CCTV 감시 대시보드</h1>
  <p className="text-gray-600 dark:text-gray-400">실시간 모니터링</p>
</div>
```

CCTV 감시 시스템은 24시간 모니터링 용도이므로 기본 다크 모드를 적용한다.
이를 통해 운용자의 눈 피로도를 줄이고 화면 번인을 방지한다.

---

## 5. tailwind.config.js 커스터마이징

### 5-1. 프로젝트 전용 설정

```javascript
// tailwind.config.js
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
        },
        status: {
          online: '#22c55e',
          offline: '#ef4444',
          recording: '#f59e0b',
        },
      },
      spacing: {
        '18': '4.5rem',
        '88': '22rem',   // 사이드바 너비
      },
      animation: {
        'pulse-slow': 'pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite',
        'blink': 'blink 1s step-end infinite',
      },
      keyframes: {
        blink: {
          '0%, 100%': { opacity: '1' },
          '50%': { opacity: '0' },
        },
      },
      fontFamily: {
        mono: ['JetBrains Mono', 'Fira Code', 'monospace'],
      },
    },
  },
  plugins: [],
};
```

### 5-2. 커스텀 색상 활용

```tsx
// 카메라 상태 표시에 커스텀 색상 사용
<span className="text-status-online">● 온라인</span>
<span className="text-status-offline">● 오프라인</span>
<span className="text-status-recording animate-blink">● 녹화중</span>
```

---

## 6. @apply 디렉티브

반복되는 유틸리티 조합을 CSS 클래스로 추출할 수 있다.

```css
/* src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  .btn-primary {
    @apply px-4 py-2 bg-primary-600 text-white rounded-lg
           hover:bg-primary-700 transition-colors duration-200
           focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2;
  }

  .btn-danger {
    @apply px-4 py-2 bg-red-600 text-white rounded-lg
           hover:bg-red-700 transition-colors duration-200;
  }

  .card {
    @apply bg-mmp-card border border-mmp-border rounded-xl p-4;
  }

  .input-field {
    @apply w-full px-3 py-2 bg-gray-800 border border-gray-600 rounded-lg
           text-white placeholder-gray-400
           focus:outline-none focus:border-primary-500 focus:ring-1 focus:ring-primary-500;
  }
}
```

```tsx
// 추출된 클래스 사용
<button className="btn-primary">카메라 추가</button>
<button className="btn-danger">카메라 삭제</button>
<div className="card">카메라 정보</div>
<input className="input-field" placeholder="카메라 이름 입력" />
```

> **주의**: `@apply`의 남용은 Tailwind의 유틸리티 퍼스트 철학에 반한다.
> 3회 이상 동일 조합이 반복될 때만 추출을 고려한다.

---

## 7. cn() 유틸리티 함수

`clsx`와 `tailwind-merge`를 조합한 `cn()` 함수는 조건부 클래스 적용과
Tailwind 클래스 충돌 해결을 동시에 처리한다.

### 7-1. 설정

```bash
npm install clsx tailwind-merge
```

```typescript
// src/lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

### 7-2. 사용 예시

```tsx
import { cn } from '@/lib/utils';

interface CameraCardProps {
  name: string;
  status: 'online' | 'offline' | 'recording';
  selected?: boolean;
  className?: string;
}

function CameraCard({ name, status, selected, className }: CameraCardProps) {
  return (
    <div
      className={cn(
        // 기본 스타일
        'rounded-xl border p-4 transition-all duration-200 cursor-pointer',
        'bg-mmp-card border-mmp-border hover:border-gray-500',
        // 조건부 스타일
        selected && 'border-primary-500 ring-2 ring-primary-500/30',
        status === 'offline' && 'opacity-50',
        // 외부에서 주입되는 클래스
        className
      )}
    >
      <div className="flex items-center justify-between mb-2">
        <h3 className="text-white font-medium truncate">{name}</h3>
        <StatusBadge status={status} />
      </div>
      <div className="aspect-video bg-black rounded-lg overflow-hidden">
        {status !== 'offline' ? (
          <VideoPlayer />
        ) : (
          <div className="flex items-center justify-center h-full text-gray-500">
            카메라 오프라인
          </div>
        )}
      </div>
    </div>
  );
}
```

### 7-3. tailwind-merge의 충돌 해결

```typescript
// 일반 clsx: 두 클래스가 모두 적용됨 (충돌)
clsx('px-4', 'px-6');  // → "px-4 px-6" (px-6이 우선하지만 불확실)

// tailwind-merge: 후자가 이김
twMerge('px-4', 'px-6');  // → "px-6" (명확한 해결)

// cn()을 사용하면 외부에서 스타일 오버라이드 가능
<CameraCard className="p-6" />  // 기본 p-4가 p-6으로 안전하게 교체됨
```

---

## 8. 프로젝트 UI 컴포넌트 스타일링

### 8-1. 사이드바

```tsx
function Sidebar({ open }: { open: boolean }) {
  return (
    <aside
      className={cn(
        'fixed left-0 top-0 h-screen bg-mmp-card border-r border-mmp-border',
        'flex flex-col transition-all duration-300 z-20',
        open ? 'w-88' : 'w-16'
      )}
    >
      {/* 로고 영역 */}
      <div className="flex items-center h-16 px-4 border-b border-mmp-border">
        <img src="/logo.svg" alt="MMP" className="w-8 h-8" />
        {open && (
          <span className="ml-3 text-white font-bold text-lg">
            버티포트 CCTV
          </span>
        )}
      </div>

      {/* 메뉴 목록 */}
      <nav className="flex-1 py-4 overflow-y-auto">
        <SidebarItem icon={Monitor} label="대시보드" active />
        <SidebarItem icon={Camera} label="카메라 관리" />
        <SidebarItem icon={Video} label="녹화 영상" />
        <SidebarItem icon={Settings} label="설정" />
      </nav>

      {/* 시스템 정보 */}
      <div className="p-4 border-t border-mmp-border">
        <SystemStatus />
      </div>
    </aside>
  );
}
```

### 8-2. 헤더

```tsx
function Header() {
  const toggleSidebar = useLayoutStore((s) => s.toggleSidebar);
  const layout = useLayoutStore((s) => s.layout);

  return (
    <header className="h-16 bg-mmp-card border-b border-mmp-border flex items-center justify-between px-6">
      <div className="flex items-center gap-4">
        <button
          onClick={toggleSidebar}
          className="p-2 rounded-lg hover:bg-gray-700 transition-colors text-gray-400 hover:text-white"
        >
          <Menu className="w-5 h-5" />
        </button>
        <h1 className="text-white text-lg font-semibold">
          실시간 모니터링
        </h1>
      </div>

      <div className="flex items-center gap-3">
        {/* 레이아웃 선택 버튼 그룹 */}
        <div className="flex bg-gray-800 rounded-lg p-1">
          {(['1x1', '2x2', '3x3', '4x4'] as const).map((l) => (
            <button
              key={l}
              className={cn(
                'px-3 py-1.5 rounded-md text-sm transition-colors',
                layout === l
                  ? 'bg-primary-600 text-white'
                  : 'text-gray-400 hover:text-white'
              )}
            >
              {l}
            </button>
          ))}
        </div>

        {/* 날씨 정보 */}
        <WeatherWidget />

        {/* 사용자 프로필 */}
        <UserAvatar />
      </div>
    </header>
  );
}
```

### 8-3. 상태 배지 컴포넌트

```tsx
function StatusBadge({ status }: { status: 'online' | 'offline' | 'recording' }) {
  return (
    <span
      className={cn(
        'inline-flex items-center gap-1.5 px-2.5 py-0.5 rounded-full text-xs font-medium',
        {
          'bg-emerald-500/10 text-emerald-400': status === 'online',
          'bg-red-500/10 text-red-400': status === 'offline',
          'bg-amber-500/10 text-amber-400': status === 'recording',
        }
      )}
    >
      <span
        className={cn('w-1.5 h-1.5 rounded-full', {
          'bg-emerald-400': status === 'online',
          'bg-red-400': status === 'offline',
          'bg-amber-400 animate-blink': status === 'recording',
        })}
      />
      {{ online: '온라인', offline: '오프라인', recording: '녹화중' }[status]}
    </span>
  );
}
```

---

## 9. 정리

- Tailwind CSS는 유틸리티 퍼스트 방식으로 HTML 내에서 직접 스타일을 적용하며 런타임 오버헤드가 없다
- 반응형 접두사(`sm:`, `md:`, `lg:`)로 별도 미디어 쿼리 없이 반응형 레이아웃을 구현한다
- `darkMode: 'class'` 설정으로 `dark:` 접두사를 통한 다크 모드 전환이 가능하다
- `tailwind.config.js`에서 프로젝트 고유 색상, 애니메이션, 폰트를 확장하여 디자인 시스템을 구축한다
- `@apply` 디렉티브로 반복 유틸리티 조합을 CSS 클래스로 추출할 수 있으나 남용은 지양한다
- `cn()` 함수(`clsx` + `tailwind-merge`)로 조건부 스타일 적용과 클래스 충돌 해결을 처리한다
- CCTV 대시보드의 카메라 카드, 사이드바, 헤더 등 핵심 UI 컴포넌트에 일관된 다크 테마를 적용한다
