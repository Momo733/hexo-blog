---
title: 使用gorm请注意
date: 2019-06-13 17:15:43
tags:
 - gorm
 - performance
---

golang的orm框架比较流行的是gorm和xorm，gorm的封装对比于xorm来说更加完善，使用不当一不小心就能使你的服务性能下降很多，本文主要说些你使用过程中需要注意的地方。

## 1.指定数据类型大小
gorm相当便利，你只需要建立所需要的结构，使用``db.CreateTable(model)``就可以创建自己的数据库表，gorm允许你使用tag来指定数据类型。
```
type User struct {
  gorm.Model
  Name         string
  Age          sql.NullInt64
  Birthday     *time.Time
  Email        string  `gorm:"type:varchar(100);unique_index"`
  Role         string  `gorm:"size:255"` // set field size to 255
  MemberNumber *string `gorm:"unique;not null"` // set member number to unique and not null
  Num          int     `gorm:"AUTO_INCREMENT"` // set num to auto incrementable
  Address      string  `gorm:"index:addr"` // create index with name `addr` for address
  IgnoreMe     int     `gorm:"-"` // ignore this field
}
```
如果是定长的数据，类似md5加密的数据，可以使用``gorm:"type:char(32)"``，这样数据库查询将使用更小的空间，由于是定长字段，将更快的被查询到。gorm指定的string对应的数据类型默认是``varchar(255)``，对于能够模糊估计的一切字段最好确定数据的大小，当数据超出长度255，将会报错。

如果能够确定一个值的大小，最好使用``int8 int32 int64 uint8 uint32 uint64``来指定字段的值的范围，这样将更加节省空间，而不是使用``int``类型，因为根据编译的环境不同，该值的范围是不一样的，32位机器``int``值介于-2,147,483,648 到+2,147,483,647之间，如果你仅仅想要作为个年龄的字段类型，那么显然是非常的不合理，同理于在64位的机器上。

## 2.指定save()表名
在gorm里面使用了非常多的反射，使用save()将会反射你传入的结构体的名称，根据下划线命名，作为更新或者新建的表名，如果你需要指定表名，将会无用。
```
func (s *DB) Save(value interface{}) *DB {
	scope := s.NewScope(value)
	if !scope.PrimaryKeyZero() {
		newDB := scope.callCallbacks(s.parent.callbacks.updates).db
		if newDB.Error == nil && newDB.RowsAffected == 0 {
			return s.New().FirstOrCreate(value)
		}
		return newDB
	}
	return scope.callCallbacks(s.parent.callbacks.creates).db
}
```
在``s.New()``将会清除之前来自db结构体中所有的类型关联条件，因为gorm需要支持支持多种数据库，这种类似的bug只是为了支持sqlite。

**更新**：根据我的[PR](https://github.com/jinzhu/gorm/commit/321c636b9da51a621d51b938b404ccd5a131e299)最近版本的gorm版本中已经更新该函数代码。

## 3.特殊字段名
如果你的表不是使用gorm创建，或者字段名称不是遵循下划线命名法，那么你的列表字段将会被gorm使用反射解析成不同的于你数据库的字段名，这会导致错误。

使用``Column``字段tag将会解析成你指定的字段名。

另外在gorm中没有xorm的批量操作，但是据说这将会加入到gorm2.0版本中。

## 4.谨慎使用gorm.Model
gorm定义了一个简化的表模型结构体：
```
type Model struct {
	ID        uint `gorm:"primary_key"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt *time.Time `sql:"index"`
}
```
还是上面所说，字段ID定义的类似uint并不是很好的选择，另外的三个字段最好按需索取，特别说明``DeletedAt``字段是软删除标识，所以如果你想彻底删除数据表中的数据，不使用下面特定的方式，就会导致你的数据其实还是存在于数据表中，如果数据量级特别大，就会导致很多问题，所以谨慎使用。
```
// Delete record permanently with Unscoped
db.Unscoped().Delete(&order)
// DELETE FROM orders WHERE id=10;
```

