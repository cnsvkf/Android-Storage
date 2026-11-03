# 내부 파일 저장 vs Room: 저는 어떤 방식을 골라야 할지 헷갈렸습니다

> 앱에 데이터를 저장해야 할 때 가장 먼저 고민한 것이 **내부 파일 저장**과 **Room** 중 무엇을 쓸지였습니다.
> 저는 두 방식을 직접 코드로 써 보며 차이를 익혔고, 이 글에는 그때 정리한 내용을 적었습니다.
> 읽고 나면 여러분도 "내 데이터라면 어떤 방식을 골라야 하지?"를 스스로 판단할 수 있을 거라고 생각합니다.

---

## 0. 먼저 알아두면 좋은 용어

처음에는 용어가 낯설어서 아래 표를 따로 정리해 두고 봤습니다.

| 용어 | 제가 이해한 뜻 |
|---|---|
| 내부 저장소 | 앱만 열어 볼 수 있는 개인 창고입니다. 앱을 지우면 함께 사라집니다. |
| 파일(File) | 글을 적어 둔 메모지 한 장입니다. |
| 테이블(Table) | 엑셀의 시트입니다. 행(줄)과 열(칸)로 이루어집니다. |
| SQLite | 스마트폰 안에 들어 있는 작은 데이터베이스 엔진입니다. |
| Room | SQLite를 편하게 쓰도록 감싸 놓은 도구입니다. |

---

## 1. 한눈에 비교표

두 방식을 처음 비교했을 때는 이렇게 보였습니다.

| 구분 | 내부 파일 (JSON 등) | Room |
|---|---|---|
| 비유 | 서랍 하나에 노트를 통째로 넣는 느낌 | 엑셀 시트에 행별로 정리하는 느낌 |
| 한 항목만 바꿀 때 | 파일 전체를 다시 씁니다 | 그 행만 고칩니다 |
| 검색 | 전부 읽고 코드로 거릅니다 | SQL 한 줄로 찾습니다 |
| 화면 자동 갱신 | 직접 알림을 만들어야 합니다 | `Flow`로 자동 갱신됩니다 |
| 구조가 바뀔 때 | 코드로 직접 변환합니다 | `Migration`으로 관리합니다 |
| 준비 과정 | 거의 없습니다 | 어노테이션과 설정이 필요합니다 |
| 어울리는 데이터 | 설정값, 작은 목록 | 목록이 많고 검색·정렬이 필요한 데이터 |

> **제가 내린 한 줄 결론**
> 데이터가 **작고 단순**하면 파일, **많고 자주 찾아봐야** 하면 Room을 쓰면 됩니다.

---

## 2. 내부 파일 저장

### 2-1. 어디에 저장할지 먼저 골랐습니다

앱 안에는 파일을 둘 수 있는 폴더가 몇 개 있습니다. 저는 아래 세 가지를 먼저 구분했습니다.

| 폴더 | 특징 | 제가 떠올린 사용 예 |
|---|---|---|
| `filesDir` | 계속 가지고 있어야 하는 파일 (백업 대상) | 사용자 설정 |
| `noBackupFilesDir` | 계속 가지고 있지만 백업에서는 뺍니다 | 기기마다 달라야 하는 값 |
| `cacheDir` | 임시 파일입니다. 공간이 모자라면 시스템이 지울 수 있습니다 | 다시 만들 수 있는 파일 |

### 2-2. 파일을 안전하게 쓰는 방법: `AtomicFile`

파일에 그냥 쓰면 될 것 같지만, 쓰는 도중에 앱이 꺼지면 파일이 반만 쓰인 채 남을 수 있습니다. `AtomicFile`은 **임시 파일에 먼저 다 쓰고, 다 쓰면 원래 파일과 바꿔치기**하는 방식이라 이 문제를 막아 줍니다.

```kotlin
import android.content.Context
import android.util.AtomicFile
import java.io.File
import java.io.IOException

class MemoLocalStore(context: Context) {

    // 백업에서 빠지는 내부 폴더 안의 memos.json 파일을 가리킵니다.
    private val atomicFile = AtomicFile(File(context.noBackupFilesDir, "memos.json"))

    // 저장하기
    fun save(json: String) {
        val stream = atomicFile.startWrite()   // 임시 파일을 엽니다.
        try {
            stream.write(json.toByteArray(Charsets.UTF_8))
            atomicFile.finishWrite(stream)     // 성공하면 임시 파일을 진짜 파일로 바꿉니다.
        } catch (e: IOException) {
            atomicFile.failWrite(stream)       // 실패하면 임시 파일을 버리고, 기존 파일은 그대로 둡니다.
            throw e
        }
    }

    // 불러오기 (파일이 없으면 null)
    fun load(): String? {
        if (!atomicFile.baseFile.exists()) return null
        return atomicFile.readFully().toString(Charsets.UTF_8)
    }
}
```

**제가 코드를 읽은 순서**
1. `startWrite()`로 임시 파일을 엽니다.
2. 내용을 씁니다.
3. 성공하면 `finishWrite()`, 실패하면 `failWrite()`를 호출합니다.

### 2-3. 장단점 정리

- ✅ 설정 한두 개처럼 **작은 데이터**를 빠르게 저장할 수 있습니다.
- ✅ 준비할 것이 거의 없습니다.
- ❌ 항목 하나만 바꿔도 **파일 전체**를 다시 써야 합니다.
- ❌ 검색, 정렬, 필터를 전부 **직접 코드로** 만들어야 합니다.
- ❌ 여러 곳에서 동시에 고치면 충돌을 **직접** 막아야 합니다.

> 설정 값처럼 작은 데이터라면 **DataStore**도 살펴볼 만합니다. 저는 파일과 Room의 차이에만 집중했습니다.

---

## 3. Room

### 3-1. Room을 이루는 세 가지 조각

Room은 조각 세 개가 맞물려 돌아갑니다. 저는 이 세 가지만 알면 대부분의 코드를 읽을 수 있다는 것을 알게 됐습니다.

| 조각 | 하는 일 | 제가 떠올린 비유 |
|---|---|---|
| `@Entity` | 테이블 하나를 정의합니다 | 엑셀 시트의 열 제목 |
| `@Dao` | 데이터를 읽고 쓰는 함수 모음입니다 | 시트에서 찾기·고치기 버튼 |
| `@Database` | 전체 데이터베이스의 입구입니다 | 엑셀 파일 자체 |

### 3-2. 가장 작은 예제

```kotlin
// 1) Entity: "memo" 표를 정의합니다. 한 줄이 메모 한 개입니다.
@Entity(tableName = "memo")
data class Memo(
    @PrimaryKey val id: Long,                              // 행을 구분하는 번호
    @ColumnInfo(name = "title") val title: String,         // 제목 칸
    @ColumnInfo(name = "created_at") val createdAt: Long,  // 만든 시각 칸
)

// 2) Dao: 표를 다루는 함수들입니다.
@Dao
interface MemoDao {

    // 최신 메모부터 가져옵니다. Flow라서 표가 바뀌면 새 목록이 자동으로 나옵니다.
    @Query("SELECT * FROM memo ORDER BY created_at DESC")
    fun observeAll(): Flow<List<Memo>>

    // 같은 id가 있으면 고치고, 없으면 새로 추가합니다.
    @Upsert
    suspend fun upsert(memo: Memo)

    @Query("DELETE FROM memo WHERE id = :id")
    suspend fun deleteById(id: Long)
}

// 3) Database: 입구입니다. 어떤 표(entities)를 가지는지 알려 줍니다.
@Database(entities = [Memo::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun memoDao(): MemoDao
}
```

> `Flow`와 `suspend`가 처음에는 어려울 수 있습니다. 이해하는 방식은 이렇습니다. `Flow`는 "값이 바뀔 때마다 계속 알려 주는 흐름"이고, `suspend`는 "끝날 때까지 기다리는 함수"입니다.

### 3-3. 데이터베이스는 한 번만 만듭니다

데이터베이스는 앱 전체에서 **한 번만** 만드는 것이 좋다고 적혀 있었습니다. 여러 번 만들면 낭비이고 문제가 생길 수 있습니다.

```kotlin
val db = Room.databaseBuilder<AppDatabase>(
    applicationContext,
    "app-database",
)
    .setDriver(AndroidSQLiteDriver()) // Room 3.0 기준 설정입니다.
    .build()

val memoDao = db.memoDao()
```

> 저는 Room 3.0(`androidx.room3`) 문서를 기준으로 공부했습니다. 여러분이 쓰는 Room 버전이 다르면 패키지 이름과 빌더 호출이 달라질 수 있으니 본인 버전의 문서를 꼭 확인해 보세요.

### 3-4. Room의 장단점

**장점**
- 📌 **SQL 오타를 빌드할 때 잡아 줍니다.** 실행해 보고서야 에러를 아는 일이 줄어듭니다.
- 📌 **`Flow`로 화면을 자동으로 갱신**할 수 있습니다.
- 📌 **`Migration`**으로 구조가 바뀌어도 기존 데이터를 옮길 수 있습니다.

**단점**
- 📌 설정할 것이 많습니다. 어노테이션, KSP, Entity·Dao·Database 클래스가 늘어납니다.
- 📌 **구조를 바꾸면 버전과 Migration을 반드시** 챙겨야 합니다.
- 📌 데이터가 아주 적다면 과한 준비일 수 있습니다.

---

## 4. 같은 기능을 두 방식으로 비교하면

"메모를 저장하고, 제목으로 검색하고, 바뀌면 화면을 갱신한다"는 기능을 두 방식으로 나눠 보면 다음과 같습니다.

| 해야 할 일 | 파일 방식 | Room 방식 |
|---|---|---|
| 메모 저장 | 전체 목록을 JSON으로 다시 씁니다 | `upsert` 한 번이면 됩니다 |
| 제목 검색 | 목록을 읽고 `filter`로 거릅니다 | `WHERE title LIKE :keyword` |
| 화면 갱신 | 바뀔 때마다 직접 알림을 보냅니다 | `Flow`가 알아서 전달합니다 |
| 구조 변경(칸 추가) | JSON을 읽는 코드를 고치고 옛 파일도 처리합니다 | `Migration`을 작성합니다 |

> 메모가 **수십 개**라면 저는 파일로도 충분하다고 생각합니다.
> 메모가 **수천 개**이고 검색과 정렬이 많아진다면 Room이 훨씬 편하다고 느꼈습니다.

---

## 5. 선택 기준

```
1. 데이터가 작고 단순한가?
   ├─ 예  → 내부 파일 (값이 적으면 DataStore도 좋음)
   └─ 아니오 → 2번으로
2. 목록이 길고, 검색·정렬·관계가 필요한가?
   ├─ 예  → Room
   └─ 아니오 → 내부 파일도 괜찮음
3. 화면이 데이터 변화를 즉시 반영해야 하는가?
   └─ 예 → Room + Flow
```

**기억해 둘 한 문장**
> "작고 단순하면 파일, 많고 찾아봐야 하면 Room."

---

## 6. 자주 하는 실수

| 자주 하는 실수 | 이렇게 고칩니다 |
|---|---|
| 영구 데이터를 `cacheDir`에 저장함 | 영구 데이터는 `filesDir` 또는 `noBackupFilesDir`에 둡니다 |
| `AtomicFile` 없이 바로 파일에 씀 | `startWrite` · `finishWrite`를 사용 |
| Room 데이터베이스를 화면마다 새로 만듦 | 한 번만 만들고 재사용 |
| 구조를 바꾸면서 버전을 올리지 않음 | `version`을 올리고 `Migration`을 작성 |

---

## 참고 자료

- [Save data in a local database using Room (Android Developers)](https://developer.android.com/training/data-storage/room)
- [Data and file storage overview (Android Developers)](https://developer.android.com/training/data-storage)

> 조사 시점: 2026-11-02. Room 예제는 Room 3.0 공식 문서를 기준으로 작성했습니다.
