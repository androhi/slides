class: center, middle

# ActiveAndroidあれこれ
Android開発アンチパターン勉強会#1
.footnote[下川 敬弘 (@androhi)]

---
# あれこれ
:
### 1. 自己紹介
### 2. ActiveAndroidについて
### 3. ContentProvider対応
### 4. DB Migration
### 5. Android 5.0対応

---
# 自己紹介
:
### * 株式会社Zaim
### * Androidアプリ担当
### * Tech Institute サポート講師

---
# ActiveAndroidについて
:
### * ActiveRecordスタイルのORM（Object Relational Mapper）ライブラリ
### * SQLステートメントを書かずにDBにアクセスできる

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
:
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

PackageNameに依存するAuthorityを使用しているため
build.gradleでapplicationIdSuffixとか付けると
providerの参照が取れなくなってしまう

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
# DB Migration

---
# Android 5.0対応

---