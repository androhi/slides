class: center, middle

# ActiveAndroidあれこれ
Android開発アンチパターン勉強会#1
.footnote[下川 敬弘 (@androhi)]

---
# あれこれ

### 1. 自己紹介
### 2. ActiveAndroidについて
### 3. ContentProvider対応
### 4. Schema migrations
### 5. Android 5.0対応

---
# 自己紹介

### - 株式会社Zaim
### - Androidアプリ担当
### - Tech Institute サポート講師

---
# ActiveAndroidについて
### - ActiveRecordスタイルのORM（Object Relational Mapper）ライブラリ
### - SQLステートメントを書かずにDBにアクセスできる


新しいレコードをInsertする例:
```java
MyEntityClass myEntity = new MyEntityClass();
myEntity.name = "hoge";
myEntity.save();
```

---
layout: true
# ContentProvider対応

---
## ActiveAndroidでContentProviderを使う方法
- Manifestファイルに`<provider>`を定義
- ModelクラスのIDを変更
```xml
<provider android:authorities="com.example"
          android:exported="false"
          android:name="com.activeandroid.content.ContentProvider" />
```
```java
@Table(name = "my_entity", id = BaseColumns._ID)
public class MyEntityClass extends Model {
}
```
--
ModelクラスごとにContent URIを生成出来るようになる
```java
URI contentUri = ContentProvider.createUri(MyEntityClass.class, null);
```

---
## ActiveAndroidのAuthorityの使い方

[ContentProvider.java#L152](https://github.com/pardom/ActiveAndroid/blob/master/src/com/activeandroid/content/ContentProvider.java#L152)
```java
	public static Uri createUri(Class<? extends Model> type, Long id) {
		final StringBuilder uri = new StringBuilder();
		uri.append("content://");
*		uri.append(sAuthority);
		uri.append("/");
		uri.append(Cache.getTableName(type).toLowerCase());

		if (id != null) {
			uri.append("/");
			uri.append(id.toString());
		}

		return Uri.parse(uri.toString());
	}

	protected String getAuthority() {
*		return getContext().getPackageName();
	}
```

---
## こんなケースで困る

PackageNameに依存するAuthorityを使用しているため、
build.gradleでapplicationIdSuffixとか付けると
providerの参照が取れなくなってしまう。

```groovy
buildTypes {
  debug {
    applicationIdSuffix ".debug"
    versionNameSuffix "-debug"
  }
}
```

---
## 解決策
AndroidGradlePluginの[ManifestMergerのplaceholder](http://tools.android.com/tech-docs/new-build-system/user-guide/manifest-merger#TOC-Placeholder-support)の機能を使う

・修正前
```xml:AndroidManifest.xml
<provider
  android:authorities="com.example"
  android:exported="false"
  android:name=".CustomContentProvider" />
```

```java:Sample.java
public static final String CONTENT_AUTHORITY = "com.example";
```
--
・修正後
```xml:AndroidManifest.xml
<provider
  android:authorities="${applicationId}"
  android:exported="false"
  android:name=".CustomContentProvider" />
```

```java:Sample.java
public static final String CONTENT_AUTHORITY = BuildConfig.APPLICATION_ID;
```

---
layout: true
# Schema migrations

---

テーブルのカラムが追加／削除されるときは、SQLスクリプトを書いたファイルを
assetsフォルダに入れておくと、DBバージョンが上がったときに実行してくれる。
```xml
<meta-data android:name="AA_DB_NAME" android:value="Sample.db" />
<meta-data android:name="AA_DB_VERSION" android:value="1" />
```

`assets/migrations/<NewVersion>.sql`
```txt
ALTER TABLE Items ADD COLUMN color INTEGER;
```

---
## こんなケースで困る

途中から追加されたテーブルに対してmigrationを行うと、
migrationに失敗するケースがある。

1. v1.0.0 : Itemsテーブル無し
2. v1.0.1 : Itemsテーブル追加
3. v1.0.2 : Itemsテーブルにカラム追加

--

- 2 -> 3のアップデートは成功する
- 1 -> 3のアップデートは失敗する

---
## 失敗する原因

データベースの新規作成時にもmigration処理が実行されるため、
既に存在するカラムを追加しようとしてエラーになる。

[DatabaseHelper.java](https://github.com/pardom/ActiveAndroid/blob/master/src/com/activeandroid/DatabaseHelper.java)
```java
public final class DatabaseHelper extends SQLiteOpenHelper {
	...
	
	@Override
	public void onCreate(SQLiteDatabase db) {
		executePragmas(db);
		executeCreate(db);
		executeMigrations(db, -1, db.getVersion());
		executeCreateIndex(db);
	}

	@Override
	public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
		executePragmas(db);
		executeCreate(db);
		executeMigrations(db, oldVersion, newVersion);
	}
```

---
## 解決策

（いや、これバグだろというissueもあるけど…）

テーブルを作り直すようにするSQLスクリプトにする。

```txt
CREATE TABLE Temp (Id INTEGER AUTO_INCREMENT PRIMARY KEY, Column TEXT NOT NULL, Column2 INTEGER NULL);
INSERT INTO Temp (Id, Column, Column2) SELECT Id, Column, 0 FROM Entity;
DROP TABLE Items;
ALTER TABLE Temp RENAME TO Items;
```
--

ただ、これも見づらくてメンテしづらそう…。

---
## 解決策つづき

コメント＋複数行に対応したらしい（試してません）

参考元: [Added a new SQL parser for migrations. #215](https://github.com/pardom/ActiveAndroid/pull/215)

```xml
<meta-data
    android:name="AA_SQL_PARSER"
    android:value="delimited" />
```
```txt
-- Create table for migration
CREATE TABLE Temp
(
    Id INTEGER AUTO_INCREMENT PRIMARY KEY,
    Column TEXT NOT NULL,
    Column2 INTEGER NULL /* this column is new */
);

-- Migrate data
INSERT INTO Temp
(
    Id,
    Column,
    Column2
)
SELECT  Id,
        Column,
        0 -- there's no such value in the old table
        FROM Items;

-- Rename Temp to Items
DROP TABLE Items;
ALTER TABLE Temp RENAME TO Items;
```

---
layout: false

# Android 5.0対応

API 21でイニシャライズでクラッシュする問題がある
そしてまだ対応中らしい。（関連するPullReqはあるにはあるけど…）

[Problem initializing ActiveAndroid in API 21 #283](https://github.com/pardom/ActiveAndroid/issues/283)

**解決策？**

ProGuardをかけてあげると大人しくなりました…
（なぜだか理解できてません）

```txt
-libraryjars ./Zaim/libs/activeandroid.jar
-keep class com.activeandroid.** { *; }
-keep class com.activeandroid.**.** { *; }
-keep class * extends com.activeandroid.Model
-keepattributes Column
-keepattributes Table
-keepclasseswithmembers class * { @com.activeandroid.annotation.Column <fields>; }
```
---
class: middle, center
background-image: url(ebisu.jpg)

# Zaimではエンジニアを募集してます！
