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
如果是定长的数据，类似md5加密的数据，可以使用``gorm:"type:char(32)"``，这样数据库查询将使用更小的空间，``varchar``保留长度在255以内将会额外使用一个字节来存储字符串的长度，超过255则是需要两个字节来保存长度。gorm指定的string对应的数据类型默认是``varchar(255)``，对于能够模糊估计的一切字段最好确定数据的大小，当数据超出长度255，将会报错。

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

**更新**：根据我的[PR](https://github.com/jinzhu/gorm/commit/321c636b9da51a621d51b938b404ccd5a131e299)最近版本的gorm版本中已经更新该函数，修改为``s.New().Table(scope.TableName()).FirstOrCreate(value)``。

## 3.特殊字段名
如果你的表不是使用gorm创建，或者字段名称不是遵循下划线命名法，那么你的列表字段将会被gorm使用反射解析成不同的于你数据库的字段名，这会导致错误。

使用``Column``字段tag将会解析成你指定的字段名。
```
// 重设列名
type Animal struct {
    AnimalId    int64     `gorm:"column:beast_id"`         // 设置列名为`beast_id`
    Birthday    time.Time `gorm:"column:day_of_the_beast"` // 设置列名为`day_of_the_beast`
    Age         int64     `gorm:"column:age_of_the_beast"` // 设置列名为`age_of_the_beast`
}
```

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

## 5.请准确指定表名
在下面的查询中报错表名未找到，其实gorm会根据用反射你定义的destination struct name和filed name 来作为查询数据库的表名和字段名，所以在下面的这种情况下，即使指定了``Model(&model.Article{})``，并且设置了结构体的TableName()接口，最后也会导致两个结构体的名字不一样报错。
```
type Article struct {
		Id          uint32    `json:"id"`
		ArticleUuid string    `gorm:"type:varchar(100)" json:"article_uuid"`
		Title       string    `gorm:"type:varchar(50)" json:"title"`
		Body        string    `gorm:"type:text" json:"body"`
		BodyHtml    string    `gorm:"type:text" json:"body_html"`
		CreatedAt   time.Time `json:"created_at"`
}

var article []Article
if err = db.Model(&model.Article{}).Where(`1=1`).Find(&article).Error; err != nil {
    println(err.Error()) //Error 1146: Table 'naimida.articles' doesn't exist
    return
}
  ```

model.Article

```
type Article struct {
	Id          uint32    `json:"id"`
	ArticleUuid string    `gorm:"type:varchar(100)" json:"article_uuid"`
	Title       string    `gorm:"type:varchar(50)" json:"title"`
	Body        string    `gorm:"type:text" json:"body"`
	BodyHtml    string    `gorm:"type:text" json:"body_html"`
	CreatedAt   time.Time `json:"created_at"`
}

func (article Article) TableName() string {
	return "article"
}
```
grom是默认使用结构体的复数结构，所以也可以使用``db.SingularTable(true)``这样也不会使用复数去解析结构体的name，也会解决上面的的表名不存在的问题。

但是如果是两个完全不一样的结构体名称，不存着单复数之分，那么在查询的时候最好指定表名，把代码中的``Model(&model.Article{})``换成``Table("article")``就可以了。

## 6.隐藏的查询条件
在gorm中会为了简便查询的使用，会给``find()``和``first()``等函数中添加一些特殊的条件，但是往往因为我们对于数据库的操作是多变的，所以这些额外的操作在某些时候是会给我们的造成负担，所以我们非常有必要深入的了解一些函数内部对应的实现，实现我们对于自己的SQL可控。

``find()``,``first()``,``last()``这些函数如果你没有指定查询的表名，那么gorm就会把你需要刷入数据的结构体的名称按照下换线规则来作为表名，如果在查询的过程中需要使用表的别名例如实现了``Table("user AS u")``，同时结构体实现了``TableName()``接口，那么就会导致``SELECT * FROM user AS u  WHERE user AS u.`deleted_at` IS NULL``这种把别名语句作为表名的错误。

另外如果表中有``deleted_at``字段，那么就会在查询的条件中加入``deleted_at IS NULL``只查询不是被软删除的数据，如果需要查询到所有的数据就添加``Unscoped()``函数就可以了。

## 7.QueryExpr()没有warpped
在gorm中使用子查询的时候，例如`` SELECT * FROM user WHERE id = (SELECT * FROM user LIMIT 1);``子查询代表括号中的内容，但是使用``QueryExpr()``不会添加括号，最后就会导致sql语法错误。

1.在使用``QueryExpr()``把对应的参数使用括号包裹，免除语法错误

2.使用``SubQuery``作为子查询执行语法，这是对于``QueryExpr()``的括号包装

另外在gorm中没有xorm的批量操作，以及sql缓存，但是据说这将会加入到gorm2.0版本中。