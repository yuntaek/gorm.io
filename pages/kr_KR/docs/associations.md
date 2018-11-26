---
title: Associations
layout: page
---

## Auto Create/Update

GORM 은 레코드 생성 및 업데이트시 참조하고 있는 결합 관계에 있는 객체를 자동으로 저장합니다. 결합 관계에 있는 객체가 주키(Primary Key)를 가지고 있으면, GORM은 저장할 때 레코들르 업데이트 하고 그 외에는 레코들 생성합니다.

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

## 자동 생성 생략 


`AutoUpdating`이 되지 않도록 설정하여도, 주키를 가지고 있지 않는 결합레코드는 생성되고, 참조레코드는 저장이 됩니다.

To disable this, you could set DB setting `gorm:association_autocreate` to `false`

```go
// Don't create associations w/o primary key, WON'T save its reference
db.Set("gorm:association_autocreate", false).Create(&user)
db.Set("gorm:association_autocreate", false).Save(&user)
```

or use GORM tags, `gorm:"association_autocreate:false"`

```
type User struct {
  gorm.Model
  Name       string
  // Don't create associations w/o primary key, WON'T save its reference
  Company1   Company `gorm:"association_autocreate:false"`
}
```

## Skip AutoCreate/Update

To disable both `AutoCreate` and `AutoUpdate`, you could use those two settings together

```go
db.Set("gorm:association_autoupdate", false).Set("gorm:association_autocreate", false).Create(&user)

type User struct {
  gorm.Model
  Name    string
  Company Company `gorm:"association_autoupdate:false;association_autocreate:false"`
}
```

Or use `gorm:save_associations`

```
db.Set("gorm:save_associations", false).Create(&user)
db.Set("gorm:save_associations", false).Save(&user)

type User struct {
  gorm.Model
  Name    string
  Company Company `gorm:"association_autoupdate:false"`
}
```

## Skip Save Reference

If you don't even want to save association's reference when updating/saving data, you could use below tricks

```go
db.Set("gorm:association_save_reference", false).Save(&user)
db.Set("gorm:association_save_reference", false).Create(&user)
```

or use tag

```go
type User struct {
  gorm.Model
  Name       string
  CompanyID  uint
  Company    Company `gorm:"association_save_reference:false"`
}
```

## Association Mode

Association Mode contains some helper methods to handle relationship related things easily.

```go
// Start Association Mode
var user User
db.Model(&user).Association("Languages")
// `user` is the source, must contains primary key
// `Languages` is source's field name for a relationship
// AssociationMode can only works if above two conditions both matched, check it ok or not:
// db.Model(&user).Association("Languages").Error
```

### Find Associations

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
