# [Android] 내부 파일 저장 vs Room, 언제 무엇을 쓸까?

> 로컬에 데이터를 저장할 때 가장 먼저 부딪히는 선택지는 **내부 파일 저장**과 **Room**입니다.
> 이 글에서는 두 방식의 차이를 정리하고, 상황별 선택 기준을 정리합니다.

---

## 1. 한눈에 보기

| 구분 | 내부 파일 (JSON 등) | Room |
|---|---|---|
| 저장 형태 | 파일 1개 = 데이터 덩어리 | SQLite 테이블 (행 단위) |
| 쿼리 | 직접 읽고 필터링 | SQL을 `@Query`로 작성, 컴파일 시 검증 |
| 일부만 수정 | 파일 전체를 다시 씀 | `@Update`로 행 단위 수정 |
| 변경 관찰 | 직접 구현 | `Flow`로 자동 갱신 |
| 스키마 변경 | 직접 파싱·변환 | `Migration`으로 관리 |
| 학습 비용 | 낮음 | 중간 (어노테이션·KSP 설정) |
| 적합한 데이터 | 설정값, 임시 상태, 작은 목록 | 목록·관계가 있는 데이터, 검색이 많은 데이터 |

> **한 줄 요약**
> 데이터가 **작고 단순**하면 파일, **많고 조회가 복잡**하면 Room을 씁니다.

---

## 2. 내부 파일 저장

앱 내부 저장소는 **해당 앱만 접근**할 수 있는 공간입니다. 주로 아래 디렉터리를 씁니다.

| 디렉터리 | 특징 |
|---|---|
| `filesDir` | 앱이 유지해야 하는 파일. 백업 대상 |
| `noBackupFilesDir` | 앱이 유지해야 하지만 **백업에서 제외**할 파일 |
| `cacheDir` | 임시 파일. 시스템이 공간 부족 시 지울 수 있음 |

### 2-1. JSON 파일로 쓰기 (`AtomicFile`)

파일을 쓰다가 앱이 꺼지면 파일이 깨질 수 있습니다. `AtomicFile`은 **임시 파일에 먼저 쓰고 마지막에 교체**하는 방식이라 이 문제를 줄여 줍니다.

```kotlin
import android.content.Context
import android.util.AtomicFile
import java.io.File
import java.io.IOException

class MemoLocalStore(context: Context) {

    // 백업에서 제외되는 내부 저장소 경로
    private val atomicFile = AtomicFile(File(context.noBackupFilesDir, "memos.json"))

    // 저장: 임시 파일에 쓴 뒤 교체한다
    fun save(json: String) {
        val stream = atomicFile.startWrite()
        try {
            stream.write(json.toByteArray(Charsets.UTF_8))
            atomicFile.finishWrite(stream) // 성공하면 실제 파일로 교체
        } catch (e: IOException) {
            atomicFile.failWrite(stream)   // 실패하면 임시 파일을 버림
            throw e
        }
    }

    // 읽기: 파일이 없으면 null
    fun load(): String? {
        if (!atomicFile.baseFile.exists()) return null
        return atomicFile.readFully().toString(Charsets.UTF_8)
    }
}
```

**장점**
- 설정, 로그인 상태처럼 값이 적은 데이터를 빠르게 저장할 수 있습니다.
- 별도 설정이 필요 없습니다.

**단점**
- 한 항목만 바꿔도 **파일 전체**를 다시 써야 합니다.
- 검색·정렬·필터를 전부 코드로 직접 구현해야 합니다.
- 여러 곳에서 동시에 쓰면 **동기화를 직접** 맞춰야 합니다.

> 값 하나씩 저장한다면 **DataStore**도 좋은 선택지입니다. 이 글에서는 파일과 Room 비교에 집중합니다.

---

## 3. Room

Room은 SQLite 위에 얹힌 **추상화 계층**입니다. Android 공식 문서도 SQLite API를 직접 쓰기보다 Room을 권장합니다.

Room의 핵심 구성 요소는 3가지입니다.

| 구성 요소 | 역할 |
|---|---|
| `@Entity` | 테이블 1개 = 데이터 클래스 1개 |
| `@Dao` | 쿼리 함수 모음 (조회·삽입·수정·삭제) |
| `@Database` | DB의 진입점. Dao를 꺼내 쓰는 곳 |

### 3-1. 최소 코드

```kotlin
// 1) Entity: 테이블 정의
@Entity(tableName = "memo")
data class Memo(
    @PrimaryKey val id: Long,
    @ColumnInfo(name = "title") val title: String,
    @ColumnInfo(name = "created_at") val createdAt: Long,
)

// 2) Dao: 쿼리 정의
@Dao
interface MemoDao {

    // Flow로 반환하면 데이터가 바뀔 때마다 자동으로 새 값이 흘러나온다
    @Query("SELECT * FROM memo ORDER BY created_at DESC")
    fun observeAll(): Flow<List<Memo>>

    @Upsert
    suspend fun upsert(memo: Memo)

    @Query("DELETE FROM memo WHERE id = :id")
    suspend fun deleteById(id: Long)
}

// 3) Database: 진입점
@Database(entities = [Memo::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun memoDao(): MemoDao
}
```

생성은 보통 앱에서 **한 번만** 합니다(싱글턴).

```kotlin
val db = Room.databaseBuilder<AppDatabase>(
    applicationContext,
    "app-database",
)
    .setDriver(AndroidSQLiteDriver()) // Room 3.0 기준 드라이버 설정
    .build()

val memoDao = db.memoDao()
```

> 이 예시는 Room 3.0 문서(`androidx.room3`) 기준입니다. 프로젝트 버전에 따라 패키지와 빌더 호출이 다를 수 있으니 사용 중인 버전 문서를 함께 확인하세요.

### 3-2. Room의 장점

- **컴파일 시 SQL 검증**: 쿼리에 오타가 있으면 빌드 단계에서 잡힙니다.
- **Flow 관찰**: 화면은 DB 변경을 구독하기만 하면 됩니다.
- **마이그레이션**: 스키마가 바뀌면 `Migration`으로 기존 데이터를 옮길 수 있습니다.
- **보일러플레이트 감소**: 커서 처리와 행 변환 코드를 직접 짤 필요가 없습니다.

### 3-3. Room의 단점

- 어노테이션, KSP 설정, Entity·Dao·Database 클래스가 늘어납니다.
- **스키마를 바꾸면 버전 관리와 마이그레이션을 반드시** 챙겨야 합니다.
- 데이터가 아주 작으면 과한 구조가 될 수 있습니다.

---

## 4. 같은 기능을 두 방식으로 비교하기

"메모 목록을 저장하고, 제목 검색을 하고, 변경되면 화면을 갱신한다"는 요구를 생각해 봅니다.

| 요구 | 파일 | Room |
|---|---|---|
| 메모 저장 | 전체 목록을 JSON으로 다시 씀 | `upsert` 한 번 |
| 제목 검색 | 목록을 읽어 `filter` | `WHERE title LIKE :keyword` |
| 화면 갱신 | 변경 때마다 직접 알림 | `Flow`가 자동으로 새 값 전달 |
| 스키마 변경(필드 추가) | JSON 파싱 로직 수정 + 기존 파일 호환 처리 | `Migration` 작성 |

> 메모가 수십 개라면 파일 방식도 충분합니다. 수천 개 이상이고 검색과 정렬이 많아지면 Room이 훨씬 편합니다.

---

## 5. 선택 기준 정리

```
데이터가 작고 단순한가?
├─ 예 → 내부 파일 (또는 DataStore)
└─ 아니오
   ├─ 목록·관계·검색이 많은가? → Room
   └─ 여러 화면에서 실시간 반영이 필요한가? → Room + Flow
```

**정리**
- 설정값, 토큰처럼 작은 값 → **DataStore** 또는 파일
- 임시 전송 큐, 작은 JSON 상태 → **파일** (`AtomicFile`로 깨짐 방지)
- 목록·검색·관계가 있는 로컬 데이터 → **Room**

---

## 참고 자료

- [Save data in a local database using Room (Android Developers)](https://developer.android.com/training/data-storage/room)
- [Data and file storage overview (Android Developers)](https://developer.android.com/training/data-storage)

> 조사 시점: 2026-11-02. 예제는 공식 문서의 Room 3.0 설정을 기준으로 작성했습니다.
