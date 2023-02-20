# DynamoDB

DynamoDB是AWS提供的一种全托管NoSQL数据库服务，提供快速而可预测的的性能，能够实现无缝的拓展。
它具有以下特性：

1. 无需操作的分布式数据管理
2. 按需备份和还原
3. 项目设置TTL过期
4. 多区复制
5. 静态加密

## 基础概念

DynamoDB基本有以下三个概念，**表**，**项目**和**属性**。

* 表：与其他数据库系统类似，DynamoDB在表中存储数据。一个表是一个数据的集合
* 项目：每个表包含零个或多个项目。一个项目是一组属性，在所有其他项目中是唯一可识别的
* 属性：每个项目都是由一个或多个属性组成。一个属性是一个基本的数据元素，是不需要进一步细分的东西

### 分区键和排序键

创建一个表的时候，DynamoDB需要设计的两个重要的属性，是分区键(必选)和排序键（可选）：

* 分区键：决定数据应该所在的分区属性列，因为DynamoDB默认支持分布式，所以此项必选
* 排序键：排序键，是决定数据在分区内排序，可选

> 当分区键和排序键同时存在时，则使用二者组合成为唯一主键；当只有分区键时，则分区键作为唯一索引。

### 全局表

全局表是DynamoDB实现的多区复制实现，可以在AWS的多个地区之间互相复制，而各区之间的程序只需要访问它所在分区的副本就可以了。

### 全局二级索引

如果对数据查询，无法使用到主键组合（分区键+排序键），那么查询的效率则退化成了扫表。整体的性能是很差的，针对特定的查询条件，建立全局二级索引的方式，让基表的主键属性始终投影到某个索引。

## AWS SDK

AWS SDK 是AWS提供的一整套操作工具的集合，其中包括对DynomoDB的操作使用。编程开发阶段，基本都是使用SDK来调用对应服务。目前支持.NET、C++、Go、Java、JavaScript、Kotlin、PHP、Python、Ruby、Rust等语言。
使用SDK操作DynamoDB，需要注意，必须在系统环境变量中，增加三个环境变量：

* AWS_ACCESS_KEY_ID：AWS派发的用户Access权限ID
* AWS_SECRET_ACCESS_KEY: 与AWS_ACCESS_KEY_ID组成验证密钥
* AWS_REGION：区域，需要访问的副本所在的区域，值是AWS各区域的命名

### 初始化

使用之前，需要实例化使用上面设置的环境变量。实例化一个DynamoDB的客户端实例：

```Golang

import (
	"context"

	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
)

// BatchWriteMax 批量写入项目数量限制，以此为限，批量写入改为并行写入
const BatchWriteMax = 25

// GetClient 获取daynamoDB的客户端实例;
// 必须在环境变量中设置:
// - AWS_ACCESS_KEY_ID
// - AWS_SECRET_ACCESS_KEY
// - AWS_REGION
func GetClient(ctx context.Context) (*dynamodb.Client, error) {
	cfg, err := config.LoadDefaultConfig(ctx)
	if err != nil {
		return nil, err
	}

	_dynamoDB := dynamodb.NewFromConfig(cfg)
	return _dynamoDB, nil
}
```

### 插入数据

插入数据只需要提供**表名**，以及将数据进行转换，便可：

```Golang
    item, err := attributevalue.MarshalMap(data)
	if err != nil {
		return err
	}

	_, err = m.DB.PutItem(ctx, &dynamodb.PutItemInput{
		TableName: aws.String(UserTaskTableName),
		Item:      item,
	})
```

需要注意的是，DynamoDB使用的动态表结构，所以无需设置表结构字段，但是，必须要有设定的组建字段(即分区键和排序键)。同时，如果没有指定数据结构体的定义，DynamoDB会**将结构体属性名，直接转换为列名**。可以通过设置`dynamodbav`的Struct Tag来指定：

```Golang
	UserTask struct {
		ID               int64  `bson:"_id" json:"id" dynamodbav:"id"`                                                // 序列ID
		UserID           int64  `bson:"user_id" json:"user_id" dynamodbav:"user_id"`                                  // 玩家ID
		CreateTime       int64  `bson:"create_time" json:"create_time" dynamodbav:"create_time"`                      //  创建时间
	}
```

### 批量插入数据

批量插入数据，与插入数据不同的最大的不同是，DynamoDB限制每个请求可以插入的最大的数据条目数量，**25条**。这是一个不可用户自定义的参数（和AWS官方确认过）。因此，再批量插入时，需要使用对数目进行拆解成多个并行的25条请求：

```Golang
	t := int(math.Ceil(float64(len(data)) / float64(dynamodbx.BatchWriteMax)))
	var waitGroup sync.WaitGroup
	var err error
	for i := 0; i < t; i++ {
		waitGroup.Add(1)
		go func(i int) {
			defer waitGroup.Done()
			start := i * dynamodbx.BatchWriteMax
			end := (i + 1) * dynamodbx.BatchWriteMax
			if end > len(data) {
				end = len(data)
			}
			if _err := m.insertMany(ctx, data[start:end]); _err != nil {
				err = _err
			}
		}(i)
	}

	waitGroup.Wait()
```

### 修改数据

因为DynamoDB是一个天然支持分布式数据库，所以对项目的修改，必须使用到主键（因为数据可能分到各个块）。因此，意味着，不可以使用模糊条件，修改大量数据的情况，需要再设计上进行规避：

```Golang
	idAttr, err := attributevalue.MarshalMap(map[string]interface{}{
		"user_id": userID,
		"id":      id,
	})
	if err != nil {
		return err
	}

	var updateAttr = make(map[string]types.AttributeValueUpdate)

	for k := range cols {
		colAttr, err := attributevalue.Marshal(cols[k])
		if err != nil {
			return err
		}
		updateAttr[k] = types.AttributeValueUpdate{
			Action: types.AttributeActionPut,
			Value:  colAttr,
		}
	}

	_, err = m.DB.UpdateItem(ctx, &dynamodb.UpdateItemInput{
		TableName:        aws.String(UserTaskTableName),
		Key:              idAttr,
		AttributeUpdates: updateAttr,
	})
```

### 查询数据

查询数据，可以使用`GetItem`设定，但是只能查询主键查询。模糊查询，可以使用类SQL查询：

```Golang
	var paramsI []interface{}
	var whereCol []string

	for k := range filter {
		paramsI = append(paramsI, filter[k])
		whereCol = append(whereCol, k+"=?")
	}

	params, err := attributevalue.MarshalList(paramsI)
	if err != nil {
		return 0, err
	}
	statement := fmt.Sprintf("SELECT id FROM \"%v\" WHERE %s", UserTaskTableName, strings.Join(whereCol, " and "))

	resp, err := m.DB.ExecuteStatement(ctx, &dynamodb.ExecuteStatementInput{
		Statement:  aws.String(statement),
		Parameters: params,
	})

	if err != nil {
		return 0, err
	}
```