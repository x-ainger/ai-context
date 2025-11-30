# Code Patterns

## API Spec

### Basic
```api
type CreateRequest {
    Email string `json:"email" validate:"required,email"`
}
type CreateResponse {
    Id int64 `json:"id"`
}
@server(group: user)
service user-api {
    @handler create
    post /api/users (CreateRequest) returns (CreateResponse)
}
```

### JWT Protected
```api
@server(
    jwt: Auth
    group: user
)
service user-api {
    @handler getProfile
    get /api/users/profile returns (ProfileResponse)
}
```

### Path Variable
```api
type GetRequest {
    Id int64 `path:"id"`
}
@server(group: user)
service user-api {
    @handler get
    get /api/users/:id (GetRequest) returns (UserResponse)
}
```

## Handler

```go
func (l *Logic) Handler(req *types.Request) (*types.Response, error) {
    // Validate
    if err := l.svcCtx.Validator.StructCtx(l.ctx, req); err != nil {
        return nil, errorx.NewCodeError(400, err.Error())
    }

    // Business logic
    result, err := l.svcCtx.Model.FindOne(l.ctx, req.Id)
    if err != nil {
        return nil, errorx.NewCodeError(500, err.Error())
    }

    return &types.Response{Data: result}, nil
}
```

## Service Context

```go
type ServiceContext struct {
    Config    config.Config
    UserModel model.UserModel
    Validator *validator.Validate
}

func NewServiceContext(c config.Config) *ServiceContext {
    conn := sqlx.NewMysql(c.MySQL.DataSource)
    return &ServiceContext{
        Config:    c,
        UserModel: model.NewUserModel(conn, c.Cache),
        Validator: validator.New(),
    }
}
```

## Config

```go
type Config struct {
    rest.RestConf
    MySQL struct {
        DataSource string
    }
    Auth struct {
        AccessSecret string
        AccessExpire int64
    }
    Cache cache.CacheConf
}
```

```yaml
Name: user-api
Port: 8888
MySQL:
  DataSource: root:pass@tcp(localhost:3306)/db
Auth:
  AccessSecret: secret
  AccessExpire: 86400
Cache:
  - Host: localhost:6379
```

## Errors

```go
// types/errors.go
var (
    ErrInvalidInput = errorx.NewCodeError(400, "invalid input")
    ErrNotFound     = errorx.NewCodeError(404, "not found")
    ErrInternal     = errorx.NewCodeError(500, "internal error")
)

// Use
if err == model.ErrNotFound {
    return nil, types.ErrNotFound
}
```

## Validation

```go
type Request struct {
    Email    string `validate:"required,email"`
    Password string `validate:"required,min=8"`
    Age      int    `validate:"min=18,max=100"`
}
```

Tags: `required`, `email`, `min=X`, `max=X`, `len=X`, `oneof=a b`, `url`, `uuid`

## Database

```go
// Find one
user, err := l.svcCtx.UserModel.FindOne(l.ctx, id)

// Insert
result, err := l.svcCtx.UserModel.Insert(l.ctx, &model.User{...})

// Update
err := l.svcCtx.UserModel.Update(l.ctx, &model.User{...})

// Delete
err := l.svcCtx.UserModel.Delete(l.ctx, id)
```

## JWT

```go
// Generate
token, err := l.getJwtToken(
    l.svcCtx.Config.Auth.AccessSecret,
    time.Now().Unix(),
    l.svcCtx.Config.Auth.AccessExpire,
    userId,
)

// Extract in handler
userId := l.ctx.Value("userId").(json.Number).Int64()
```

## Generation Rules

- Always validate input
- Always pass context
- Use errorx.NewCodeError for errors
- Add validation tags
- Generate complete logic, not stubs
