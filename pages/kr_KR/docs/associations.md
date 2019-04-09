---
title: Associations
layout: page
---

## 자동 생성/업데이트

GORM 은 레코드 생성 및 업데이트시 참조하고 있는 결합 관계에서 객체를 자동으로 저장합니다. 결합 관계에 있는 객체가 주키(Primary Key)를 가지고 있으면, GORM은 저장할 때 레코들를 업데이트 하고 그 외에는 레코들 생성합니다.

```go
user := User{
	Name:            "jinzhu",
	BillingAddress:  Address{Address1: "Billing Address - Address 1"},
	ShippingAddress: Address{Address1: "Shipping Address - Address 1"},
	Emails:          []Email{
		{Email: "jinzhu@example.com"},
		{Email: "jinzhu-2@example@example.com"},
	},
	Languages:       []Language{
		{Name: "ZH"},
		{Name: "EN"},
	},
}

db.Create(&user)
//// BEGIN TRANSACTION;
//// INSERT INTO "addresses" (address1) VALUES ("Billing Address - Address 1");
//// INSERT INTO "addresses" (address1) VALUES ("Shipping Address - Address 1");
//// INSERT INTO "users" (name,billing_address_id,shipping_address_id) VALUES ("jinzhu", 1, 2);
//// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu@example.com");
//// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu-2@example.com");
//// INSERT INTO "languages" ("name") VALUES ('ZH');
//// INSERT INTO user_languages ("user_id","language_id") VALUES (111, 1);
//// INSERT INTO "languages" ("name") VALUES ('EN');
//// INSERT INTO user_languages ("user_id","language_id") VALUES (111, 2);
//// COMMIT;

db.Save(&user)
```

## 자동 업데이트 생략

이전에 있던 데이터베이스에 결합 관계설정이 있다면, 결합레코드들에 대한 자동 업데이트가 되는것을 원치 않을 수 있습니다. 
이때 DB 세팅에 `gorm:association_autoupdate` 를 `false` 로 지정하면 됩니다.


```go
// 주 키(Primary Key)를 가진 결합레코드를 업데이트 하지 말고, 참조 레코드만 저장하라
db.Set("gorm:association_autoupdate", false).Create(&user)
db.Set("gorm:association_autoupdate", false).Save(&user)
```

GORM의 tags를 이용하는 방법이 있습니다. `gorm:"association_autoupdate:false"`

```go
type User struct {
  gorm.Model
  Name       string
  CompanyID  uint
  // 주 키(Primary Key)를 가진 결합들을 업데이트 하지 말고, 참조 레코드만 저장하라
  Company    Company `gorm:"association_autoupdate:false"`
}
```

## 자동생성(AutoCreate) 생략

`AutoUpdating`이 되지 않도록 설정하여도, 주키를 가지고 있지 않는 결합레코드는 생성되고, 참조레코드는 저장이 됩니다.
자동 생성을 비화설화 하기 위해서는 DB 세팅에서  `gorm:association_autocreate` 를 `false` 로 지정하거나

```go
// Don't create associations w/o primary key, WON'T save its reference
db.Set("gorm:association_autocreate", false).Create(&user)
db.Set("gorm:association_autocreate", false).Save(&user)
```

GORM 태그를 이용하여  `gorm:"association_autocreate:false"` 할 수 있습니다.

```
type User struct {
  gorm.Model
  Name       string
  // 주키가 있든 없든 참조 레코드를 생성하지 말고, 그것의 참조 레코드도 저장하지 않을 것이다.
  Company1   Company `gorm:"association_autocreate:false"`
}
```

## 자동 생성/업데이트 생략

 `AutoCreate` 와 `AutoUpdate`를 비활성화 하기 위해 ,두개의 setting을 함께 사용할 수 있다.

```go
db.Set("gorm:association_autoupdate", false).Set("gorm:association_autocreate", false).Create(&user)

type User struct {
  gorm.Model
  Name    string
  Company Company `gorm:"association_autoupdate:false;association_autocreate:false"`
}
```

또는  `gorm:save_associations`를 사용해라

```go
db.Set("gorm:save_associations", false).Create(&user)
db.Set("gorm:save_associations", false).Save(&user)

type User struct {
  gorm.Model
  Name    string
  Company Company `gorm:"association_autoupdate:false"`
}
```

## 참조 레코들을 저장하는거 생략

만약 결합레코드의 참조 레코드가 저장되지 않기을 원한다면, 아래의 방법을 사용할 수 있다.

```go
db.Set("gorm:association_save_reference", false).Save(&user)
db.Set("gorm:association_save_reference", false).Create(&user)
```

또는 태그를 사용할 수 있다.

```go
type User struct {
  gorm.Model
  Name       string
  CompanyID  uint
  Company    Company `gorm:"association_save_reference:false"`
}
```

## Assiciation Mode(결합 모드)

Assocaiantion Mode는 관련된 객체들간에 관계를  쉡게 다룰 수 있는 헬퍼 메소드를 포함하고 있다.

```go
// Association Mode 시작
var user User
db.Model(&user).Association("Languages")
// `user` 는 소스이고, 주키를 반듯이 포함 한다.
// `Languages` 는 괜계를 위한 소스의 필드 이름이다.
// AssociationMode는 위의 두 조건이 모두 충족할때 동작한다. 에러가 있는지 잘 동작했는 확인한다.:
// db.Model(&user).Association("Languages").Error
```

### 결합관계를 찾기

Find matched associations

```go
db.Model(&user).Association("Languages").Find(&languages)
```

### Append Associations

Append new associations for `many to many`, `has many`, replace current associations for `has one`, `belongs to`

```go
db.Model(&user).Association("Languages").Append([]Language{languageZH, languageEN})
db.Model(&user).Association("Languages").Append(Language{Name: "DE"})
```

### Replace Associations

Replace current associations with new ones

```go
db.Model(&user).Association("Languages").Replace([]Language{languageZH, languageEN})
db.Model(&user).Association("Languages").Replace(Language{Name: "DE"}, languageEN)
```

### Delete Associations

Remove relationship between source & argument objects, only delete the reference, won't delete those objects from DB.

```go
db.Model(&user).Association("Languages").Delete([]Language{languageZH, languageEN})
db.Model(&user).Association("Languages").Delete(languageZH, languageEN)
```

### Clear Associations

Remove reference between source & current associations, won't delete those associations

```go
db.Model(&user).Association("Languages").Clear()
```

### Count Associations

Return the count of current associations

```go
db.Model(&user).Association("Languages").Count()
```
