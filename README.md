## Kratos-single-template

## 说明

该项目默认集成了以下的组件

- dockerfile
- docker compose
- sqlc
- casdoor
- otel
- postgres
- consul配置中心
- consul注册中心
- jwt
- minio

## 数据库

该项目不使用`ORM`, 而是使用更底层的SQL生成工具`sqlc`, 其性能损耗比 ORM 开销小, 性能接近原生 sql
sqlc 支持下列数据库

- postgres或基于 postgres 的数据
- sqlite
- mysql

### 先决条件

1. 你需要在编写项目的机器上安装 `sqlc`, 官网链接: https://docs.sqlc.dev/en/stable/overview/install.html
2. 你需要安装对应的数据库

-
- sqlc安装
  brew:

```bash
brew install sqlc
```

go:

```bash
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
```

- 数据库
- 基于postgres的分布式插件 citus:

```bash
docker compose -f /infrastructure/citus up -d
```

### 使用

与 ORM最直观的区别是SQL 生成工具是根据 `SQL语句`来生成对应的 CURD 方法, 它的执行性能由开发者手动控制, 直接操作 SQL,
需要开发者熟悉 SQL 才可以编写
而 ORM 是提供了 Go 的CURD 方法, 隐藏了 SQL 实现, 但可能对特定的 SQL 有优化. 不需要开发者熟悉 SQL 才可以编写. 使用简单,
调式复杂, 写的好坏都不知道, 写10个循环遍历千次查询都有可能, 心智负担大

sql生成有几个工具:

- sqlc
- sqlx
  本项目使用 sqlc
  使用

1. 编写 sql风格的 sqlc代码:
   语法:
   -- name: <function_name> :return_type

- <function_name>: 定义这个 sql 语句的名称, 根据这个名称来生成对应名称的方法
- return_type
    - :one 单条返回值, 返回单结构体
    - :many 多条返回值, 返回的是`切片类型`的结构体
    - :exec 无返回值
$1, $2 这些是占位符, 会被 sqlc 解析并替换
也可以使用具名参数, 如 @name
```sql
-- name: CreatePayment :one
INSERT INTO payment.payments(snowflake_id, owner, name, amount, order_id, credit_card_number, credit_card_cvv,
                             credit_card_expiration_year, credit_card_expiration_month, status)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10) RETURNING *;
```

2. 生成 Go 代码:
   在 Makefile 添加并执行`make sqlc`

# 生成sql代码
```Makefile
.PHONY: sqlc
sqlc:
	sqlc generate
```

或直接执行`sqlc generate`
生成出的go 代码在配置文件里定义的`models`目录中
```
const CreatePayment = `-- name: CreatePayment :one
INSERT INTO payment.payments(snowflake_id, owner, name, amount, order_id, credit_card_number, credit_card_cvv,
                             credit_card_expiration_year, credit_card_expiration_month, status)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
RETURNING id, snowflake_id, owner, name, amount, order_id, credit_card_number, credit_card_cvv, credit_card_expiration_year, credit_card_expiration_month, status, created_at
`
```

```go
package models

type CreatePaymentParams struct {
	SnowflakeID               int64   `json:"snowflakeID"`
	Owner                     string  `json:"owner"`
	Name                      string  `json:"name"`
	Amount                    float64 `json:"amount"`
	OrderID                   int32   `json:"orderID"`
	CreditCardNumber          string  `json:"creditCardNumber"`
	CreditCardCvv             string  `json:"creditCardCvv"`
	CreditCardExpirationYear  string  `json:"creditCardExpirationYear"`
	CreditCardExpirationMonth string  `json:"creditCardExpirationMonth"`
	Status                    string  `json:"status"`
}
```

在业务里使用:

```go
package data

import (
	"payment/internal/biz"
	"payment/internal/data/models"
)

func example() {
	params := models.CreatePaymentParams{
		SnowflakeID:               pkg.SnowflakeID().Int64(),
		Owner:                     req.Owner,
		Name:                      req.Name,
		Amount:                    req.Amount,
		OrderID:                   int32(req.OrderId),
		CreditCardNumber:          req.CreditCard.Number,
		CreditCardCvv:             req.CreditCard.Cvv,
		CreditCardExpirationYear:  req.CreditCard.ExpirationYear,
		CreditCardExpirationMonth: req.CreditCard.ExpirationMonth,
		Status:                    biz.PENDING,
	}
	payment, err := p.data.db.CreatePayment(ctx, params)
	if err != nil {
		return nil, err
	}
}

```

#### 配置
注意事项:
如果数据库的 sql 类型不加`not null`非空等, 那么它的值可能是空的, go 对空值的判断支持有限, 最好都在数据库 sql 的类型中添加
  `not null`类型, 这样 sqlc 在解析你编写的 `schema` 目录下的 sql 文件时就会编译为 go 指针
  配置文件
  编写sqlc.yaml名称固定, 直接复制即可,放到微服务根目录,参数都有注释
```yaml

  version: "2"
  sql:
    - schema: "internal/data/migrate"
      queries: "internal/data/queries"
      engine: "postgresql"
      database:
      uri: ${DB_SOURCE}
      # uri: postgresql://postgres:${PG_PASSWORD}@${PG_HOST}:${PG_PORT}/simple_bank?sslmode=disable
      # uri: postgresql://postgres:postgres@localhost:5432/simple_bank?sslmode=disable
      gen:
      go:
      package: "models"
      out: "internal/data/models"
      sql_package: "pgx/v5"
      emit_db_tags: false # 如果为 true，则将 DB 标记添加到生成的结构中。默认值为 false
      emit_prepared_queries: true # 如果为 true，则包括对准备好的查询的支持。默认值为 false
      emit_interface: true # 如果为 true，则在生成的包中输出接口 Querier 。默认值为 false
      emit_exact_table_names: true # 如果为 true，则结构名称将使用复数表名称。否则，sqlc 会尝试单数化复数表名。默认值为
      false 。
      emit_empty_slices: false # 如果为 true， :many 则查询返回的切片将为空，而不是 nil 。默认值为 false 。
      emit_exported_queries: true # 如果为 true，则可以导出自动生成的 SQL 语句以供其他包访问。
      emit_json_tags: true # 如果为 true，则将 JSON 标记添加到生成的结构中。默认值为 false 。
      emit_result_struct_pointers: false # 如果为 true，则查询结果将作为指向结构的指针返回。返回多个结果的查询将作为指针切片返回。默认值为
      false 。
      emit_params_struct_pointers: false # 如果为 true，则参数将作为指向结构的指针传递。默认值为 false 。
      emit_methods_with_db_argument: false # 如果为 true，则生成的方法将接受 DBTX 参数，而不是在 *Queries 结构上存储
      DBTX。默认值为 false 。
      emit_pointers_for_null_types: true # 如果为 true，则为可为 null 列生成的类型将作为指针发出 （即 *string ） 而不是
      database/sql null 类型（即。 NullString ）。目前仅支持 PostgreSQL if sql_package is pgx/v4 或 pgx/v5 和 SQLite。默认值为
      false 。
      emit_enum_valid_method: true # 如果为 true，则在枚举类型上生成 Valid 方法，指示字符串是否为有效的枚举值。
      emit_all_enum_values: true # 如果为 true，则为每个枚举类型发出一个函数，该函数返回所有有效的枚举值。
      emit_sql_as_comment: true # 如果为 true，则将 SQL 语句作为代码块注释发出，并附加到任何现有注释。默认值为 false 。
      # build_tags: # 如果设置，则在每个生成的 Go 文件的开头添加一个 //go:build <build_tags> 指令。
      json_tags_id_uppercase: true # 如果为 true，则 json 标记中的“Id”将为大写。如果为 false，则为 camelcase。默认值为 false
      json_tags_case_style: camel # camel: 首字符小写、 pascal: 首字符大写、 snake: 蛇形 或 none 在数据库中使用列名。默认值为
      none 。
      omit_unused_structs: false # 如果 true ，sqlc 不会生成在给定包的查询中未使用的表和枚举结构。默认值为 false 。
      output_batch_file_name: batch.go # 自定义批处理文件的名称。默认值为 batch.go 。
      output_db_file_name: db.go # 自定义数据库文件的名称。默认值为 db.go 。
      output_models_file_name: models.go # 自定义模型文件的名称。默认值为 models.go 。
      output_querier_file_name: querier.go # 自定义查询器文件的名称。默认值为 querier.go 。
      output_copyfrom_file_name: copyfrom.go # 自定义 copyfrom 文件的名称。默认值为 copyfrom.go 。
      #output_files_suffix:# 如果指定，后缀将添加到生成的文件的名称中。
      query_parameter_limit: 1 # 将为 Go 函数生成的位置参数数。若要始终发出参数结构，请将其设置为 0 。默认值为 1 。
      # rename:# 自定义生成的结构字段的名称。有关使用信息，请参阅重命名字段。
      # overrides: #它是定义的集合，用于指示使用哪些类型来映射数据库类型
      overrides:
        - db_type: "timestamptz"
          go_type: "time.Time"
        - db_type: "uuid"
          go_type: "github.com/google/uuid.UUID"
```
cd <微服务根目录>
SQLC 需要知道你的数据库架构和查询才能生成代码
创建以下目录:

- models: go 结构与数据库结构的映射
- queries: sql 文件在这里编写
- migrte: 迁移, 编写数据库的 SQL, 包含索引, 约束等, 使用 go-migrate 工具来生成 初始化的 sql, 后续添加新列都使用这个工具来增加和删除
- schema:模型, 数据库的表在这里写, 仅包含表, 不包含索引, 直接从迁移目录里复制
  mkdir ./internal/data/models
  mkdir ./internal/data/queries
  mkdir ./internal/data/migrte
  mkdir ./internal/data/schema
  事务
  添加对事务的支持: 编写`store.go 放在 `./internal/data/models` 目录下
  package models

```go
package models

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

type Store interface {
	Querier
}

type SQLStore struct {
	*Queries
	db *pgxpool.Pool
}

func NewStore(db *pgxpool.Pool) Store {
	return &SQLStore{
		db:      db,
		Queries: New(db),
	}
}

// execTx 通用的事务方法, 通过外部传递函数作为事务的运行内容
func (s *SQLStore) execTx(ctx context.Context, fn func(*Queries) error) error {
	// 开始一个事务, 如sql的begin
	tx, err := s.db.BeginTx(ctx, pgx.TxOptions{})
	if err != nil {
		return err
	}
	// 事务的运行内容
	query := New(tx)
	err = fn(query)
	// 如果事务发生错误
	if err != nil {
		// 如果回滚发生错误, 那么合并两个错误为一个错误返回回去
		if rbErr := tx.Rollback(ctx); rbErr != nil {
			return fmt.Errorf("tx err is: '%s', rellback err is: '%s'", err, rbErr)
		}
		// 返回事务错误
		return err
	}

	// 没有错误则提交该事务
	return tx.Commit(ctx)
}

```

#### 部署

## S3 store

### 先决条件

你需要在机器上部署:

- minio

#### 部署

- Docker
  建议在 Linux 部署, 如果是本机部署, 请修改`install.sh`的数据存储路径

```bash
git clone --depth 1 https://github.com/Mandala-lab/docker-deploy.git
cd docker-deploy/minio/
cat ./install.sh
chmod +x ./install.sh
./install.sh
```

- Kubernetes部署
  请查看并确认配置文件是否符合你的实际情况

```bash
git clone --depth 1 https://github.com/Mandala-lab/cloud-native-deploy.git
cd minio/helm/
cat ./operator/install.sh
cat ./tenant/install.sh
````

### 使用

项目默认封装了简单的上传/下载/分享/列出文件的功能, 它在`kratos-template/pkg/oss`目录下

## CI/CD

默认使用: GitHub Actions + kustomize + ArgoCD + Kubernetes

### 先决条件

你需要在机器上部署:

- Kubernetes
- Kubernetes 上的 ArgoCD

如果你的机器的操作系统是 Ubuntu, 那么可以直接clone我的[仓库](https://github.com/Mandala-lab/Kubernetes)一键安装单机版的
Kubernetes 集群

机器无法访问 Github 的情况:

```bash
git clone --depth 1 https://github.com/Mandala-lab/Kubernetes.git
cd Kubernetes
chmod +x ./cn-base-start.sh && ./cn-base-start.sh
```

机器可以访问 Github 的情况:

```bash
git clone --depth 1 https://github.com/Mandala-lab/Kubernetes.git
cd Kubernetes
chmod +x ./base-start.sh && ./base-start.sh
```

ArgoCD也可以通过我的另一个仓库进行安装, 不过你需要查看部署的 shell 脚本, 避免错误的安装:

```bash
git clone --depth 1 https://github.com/Mandala-lab/cloud-native-deploy.git
cd cloud-native-deploy/argo/server/yaml/new
cat 01-install.sh
cat 02-init-config.sh

```
