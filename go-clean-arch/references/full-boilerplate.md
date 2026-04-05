# Full Boilerplate Reference

This file contains the complete contents for every file in a new Clean Architecture Go project. Use this when scaffolding the entire project from scratch.

## Project Overview

- **Module**: `go-clean-example` (adjust module name as needed)
- **Go version**: 1.25.1
- **Database**: MySQL 8.0 (via `github.com/go-sql-driver/mysql`)
- **Web framework**: Gin + CORS
- **DI**: Uber FX
- **Config**: Viper
- **SQL codegen**: sqlc v2
- **Logger**: `github.com/adityaeka26/go-pkg`

## File-by-file contents

### `go.mod` and `go.sum`

Initialize with `go mod init go-clean-example`, then add dependencies:
```
github.com/adityaeka26/go-pkg v0.9.9
github.com/gin-contrib/cors v1.7.7
github.com/gin-gonic/gin v1.11.0
github.com/go-sql-driver/mysql v1.9.3
github.com/shopspring/decimal v1.4.0
github.com/spf13/viper v1.19.0
go.uber.org/fx v1.24.0
```
Run `go mod tidy` to resolve transitive deps and generate `go.sum`.

### `.env.example`
```
APP_ENV=development
REST_PORT=3000
APP_VERSION=1.0
GRACEFUL_PERIOD=0
MYSQL_USER=default
MYSQL_PASSWORD=secret
MYSQL_DATABASE=erp_sales
MYSQL_ROOT_PASSWORD=root
MYSQL_DSN=default:secret@tcp(localhost:3306)/erp_sales?parseTime=true
```

### `.env.docker`
```
APP_ENV=development
REST_PORT=3000
APP_VERSION=1.0
GRACEFUL_PERIOD=0
MYSQL_USER=default
MYSQL_PASSWORD=secret
MYSQL_DATABASE=erp_sales
MYSQL_ROOT_PASSWORD=root
MYSQL_DSN=default:secret@tcp(mysql_db:3306)/erp_sales?parseTime=true
```

### `.gitignore`
```
tmp/
.qwen/
server
.env
```

### `.air.toml`
```toml
[build]
cmd = "go build -buildvcs=false -o ./tmp/main ./cmd/server/server.go"
bin = "tmp/main"
```

### `Dockerfile`
```dockerfile
FROM golang:1.25 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go install golang.org/x/tools/cmd/godoc@v0.5.0
RUN go install github.com/air-verse/air@v1.52.3
```

### `docker-compose.yml`
```yaml
services:
  golang-app:
    build: .
    container_name: "golang-app"
    restart: always
    ports:
      - "8080:3000"
      - "6464:6464"
    environment:
      DB_HOST: mysql_db
      DB_USER: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      DB_NAME: ${MYSQL_DATABASE}
      DB_PORT: 3306
    depends_on:
      - mysql_db
    volumes:
      - .:/app
    command: godoc -http=:6464

  mysql_db:
    image: mysql:8.0.33
    container_name: "golang-mysql"
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

### `sqlc.yaml`
```yaml
version: "2"
sql:
  - engine: "mysql"
    queries: "internal/repository/mysql/user/user.sql"
    schema: "internal/repository/mysql/schema.sql"
    gen:
      go:
        package: "user"
        out: "internal/repository/mysql/user"
        emit_json_tags: true
        overrides:
          - db_type: "decimal"
            go_type:
              import: "github.com/shopspring/decimal"
              type: "Decimal"
```

### `config/config.go`
```go
package config

import (
	"sync"
	"sync/atomic"
	"github.com/spf13/viper"
)

var (
	once sync.Once
	cfg  atomic.Pointer[EnvConfig]
)

type EnvConfig struct {
	AppEnv         string `mapstructure:"APP_ENV"`
	AppName        string `mapstructure:"APP_NAME"`
	RestPort       string `mapstructure:"REST_PORT"`
	AppVersion     string `mapstructure:"APP_VERSION"`
	GracefulPeriod int    `mapstructure:"GRACEFUL_PERIOD"`
	DSN            string `mapstructure:"MYSQL_DSN"`
}

func GetConfig() *EnvConfig {
	once.Do(func() {
		c := loadConfig()
		cfg.Store(c)
	})
	return cfg.Load()
}

func loadConfig() *EnvConfig {
	var envCfg EnvConfig
	viper.AddConfigPath(".")
	viper.SetConfigFile(".env")
	viper.AutomaticEnv()
	if err := viper.ReadInConfig(); err != nil {
		if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
			panic(err)
		}
		// Continue with environment variables only
	}
	if err := viper.Unmarshal(&envCfg); err != nil {
		panic(err)
	}
	return &envCfg
}
```

### `internal/repository/mysql/schema.sql`

Full MySQL dump with 3 tables and seed data:

```sql
-- MySQL dump 10.13  Distrib 8.0.33, for Linux (x86_64)
-- Host: localhost    Database: erp_sales

DROP TABLE IF EXISTS `discounts`;
CREATE TABLE `discounts` (
  `discount_id` int NOT NULL AUTO_INCREMENT,
  `order_id` int NOT NULL,
  `discount_percent` decimal(5,2) NOT NULL,
  `description` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`discount_id`),
  KEY `order_id` (`order_id`),
  CONSTRAINT `discounts_ibfk_1` FOREIGN KEY (`order_id`) REFERENCES `orders` (`order_id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

LOCK TABLES `discounts` WRITE;
INSERT INTO `discounts` VALUES (1,1,10.00,'Promo on laptop'),(2,3,5.00,'Keyboard sale'),(3,4,15.00,'Bulk monitor discount');
UNLOCK TABLES;

DROP TABLE IF EXISTS `orders`;
CREATE TABLE `orders` (
  `order_id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `item` varchar(255) NOT NULL,
  `quantity` int NOT NULL,
  `price` decimal(10,2) NOT NULL,
  PRIMARY KEY (`order_id`),
  KEY `user_id` (`user_id`),
  CONSTRAINT `orders_ibfk_1` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

LOCK TABLES `orders` WRITE;
INSERT INTO `orders` VALUES (1,1,'Laptop',1,1200.00),(2,1,'Mouse',2,25.50),(3,2,'Keyboard',1,75.00),(4,3,'Monitor',2,300.00),(5,4,'Headphones',1,150.00),(6,2,'Webcam',1,90.00);
UNLOCK TABLES;

DROP TABLE IF EXISTS `users`;
CREATE TABLE `users` (
  `user_id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL,
  `email` varchar(255) NOT NULL,
  `location` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`user_id`),
  UNIQUE KEY `email` (`email`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

LOCK TABLES `users` WRITE;
INSERT INTO `users` VALUES (1,'Alice Johnson','alice@example.com','New York'),(2,'Bob Smith','bob@example.com','Los Angeles'),(3,'Charlie Brown','charlie@example.com','Chicago'),(4,'Diana Prince','diana@example.com','Miami');
UNLOCK TABLES;
```

### `internal/repository/mysql/user/user.sql`

```sql
-- name: GetUserByID :one
SELECT user_id, name, email, location FROM users
WHERE user_id = ?;

-- name: GetOrdersByUserID :many
SELECT o.order_id, o.user_id, o.item, o.quantity, o.price,
       COALESCE(SUM(d.discount_percent), 0) as discount_percent
FROM orders o
LEFT JOIN discounts d ON o.order_id = d.order_id
WHERE o.user_id = ?
GROUP BY o.order_id, o.user_id, o.item, o.quantity, o.price
ORDER BY o.order_id
LIMIT ? OFFSET ?;
```

### `internal/repository/mysql/conn.go`

```go
package mysql

import (
	"context"
	"database/sql"
	"go-clean-example/config"
	"time"
	_ "github.com/go-sql-driver/mysql"
)

func Open() (*sql.DB, error) {
	dsn := config.GetConfig().DSN
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		return nil, err
	}
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := db.PingContext(ctx); err != nil {
		_ = db.Close()
		return nil, err
	}
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(10)
	db.SetConnMaxLifetime(5 * time.Minute)
	db.SetConnMaxIdleTime(5 * time.Minute)
	return db, nil
}
```

### `internal/repository/repository.go`

```go
package repository

import (
	"context"
	"go-clean-example/internal/domain"
)

type UserRepository interface {
	GetUserInfo(ctx context.Context, userID int32) (*domain.GetUserInfoResponse, error)
	GetOrdersByUserID(ctx context.Context, userID int32, limit, offset int) (*domain.GetOrdersResponse, error)
}
```

### `internal/repository/mysql/user_repository.go`

```go
package mysql

import (
	"context"
	"database/sql"
	"go-clean-example/internal/domain"
	"go-clean-example/internal/repository"
	"go-clean-example/internal/repository/mysql/user"
)

type userRepository struct {
	query user.Queries
}

func NewUserRepository(db *sql.DB) repository.UserRepository {
	return &userRepository{query: *user.New(db)}
}

func (r *userRepository) GetUserInfo(ctx context.Context, userID int32) (*domain.GetUserInfoResponse, error) {
	usr, err := r.query.GetUserByID(ctx, userID)
	if err != nil {
		return nil, err
	}
	return &domain.GetUserInfoResponse{
		UserID:   usr.UserID,
		Name:     usr.Name,
		Email:    usr.Email,
		Location: func() string { if usr.Location.Valid { return usr.Location.String }; return "" }(),
	}, nil
}

func (r *userRepository) GetOrdersByUserID(ctx context.Context, userID int32, limit, offset int) (*domain.GetOrdersResponse, error) {
	orders, err := r.query.GetOrdersByUserID(ctx, user.GetOrdersByUserIDParams{
		UserID: userID,
		Limit:  int32(limit),
		Offset: int32(offset),
	})
	if err != nil {
		return nil, err
	}
	ordersResponse := make([]*domain.Order, 0, len(orders))
	for _, o := range orders {
		ordersResponse = append(ordersResponse, &domain.Order{
			UserID:          int32(o.UserID),
			OrderID:         int32(o.OrderID),
			Item:            o.Item,
			Quantity:        int32(o.Quantity),
			Price:           o.Price,
			DiscountPercent: o.DiscountPercent,
		})
	}
	return &domain.GetOrdersResponse{Orders: ordersResponse}, nil
}
```

### `internal/domain/user_domain.go`

```go
package domain

import "github.com/shopspring/decimal"

type Order struct {
	OrderID         int32           `json:"order_id"`
	UserID          int32           `json:"user_id"`
	Item            string          `json:"item"`
	Quantity        int32           `json:"quantity"`
	Price           decimal.Decimal `json:"price"`
	DiscountPercent decimal.Decimal `json:"discount_percent"`
}

func (d *Order) CalculateFinalPrice() decimal.Decimal {
	percent := d.DiscountPercent
	if percent.LessThan(decimal.Zero) {
		percent = decimal.Zero
	}
	if percent.GreaterThan(decimal.NewFromInt(100)) {
		percent = decimal.NewFromInt(100)
	}
	total := decimal.NewFromInt(int64(d.Quantity)).Mul(d.Price)
	discountFactor := decimal.NewFromInt(100).Sub(percent).Div(decimal.NewFromInt(100))
	return total.Mul(discountFactor)
}

type GetOrdersResponse struct {
	Orders []*Order `json:"orders"`
}

type GetUserInfoResponse struct {
	UserID   int32  `json:"user_id"`
	Name     string `json:"name"`
	Email    string `json:"email"`
	Location string `json:"location"`
}
```

### `internal/dto/user_dto.go`

```go
package dto

import "github.com/shopspring/decimal"

type UserResponse struct {
	UserID   int32   `json:"user_id"`
	Name     string  `json:"name"`
	Email    string  `json:"email"`
	Location string  `json:"location"`
	Orders   []Order `json:"orders"`
}

type Order struct {
	UserID     int32           `json:"user_id"`
	Item       string          `json:"item"`
	Quantity   int32           `json:"quantity"`
	Price      decimal.Decimal `json:"price"`
	Discount   decimal.Decimal `json:"discount"`
	FinalPrice decimal.Decimal `json:"final_price"`
}
```

### `internal/usecase/usecase.go`

```go
package usecase

import (
	"context"
	"go-clean-example/internal/dto"
)

type UserUsecase interface {
	GetUserWithOrders(ctx context.Context, id int32, limit, offset int) (*dto.UserResponse, error)
}
```

### `internal/usecase/user_usecase.go`

```go
package usecase

import (
	"context"
	"time"
	"go-clean-example/internal/dto"
	"go-clean-example/internal/repository"
	"github.com/adityaeka26/go-pkg/logger"
)

type userUsecase struct {
	logger         *logger.Logger
	userRepository repository.UserRepository
}

func NewUserUsecase(logger *logger.Logger, userRepo repository.UserRepository) UserUsecase {
	return &userUsecase{
		logger:         logger,
		userRepository: userRepo,
	}
}

func (u *userUsecase) GetUserWithOrders(ctx context.Context, userID int32, limit, offset int) (*dto.UserResponse, error) {
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	user, err := u.userRepository.GetUserInfo(ctx, userID)
	if err != nil {
		return nil, err
	}
	orders, err := u.userRepository.GetOrdersByUserID(ctx, user.UserID, limit, offset)
	if err != nil {
		return nil, err
	}
	ordersResp := make([]dto.Order, len(orders.Orders))
	for i, order := range orders.Orders {
		finalPrice := order.CalculateFinalPrice()
		ordersResp[i] = dto.Order{
			UserID:     order.UserID,
			Item:       order.Item,
			Quantity:   order.Quantity,
			Price:      order.Price,
			Discount:   order.DiscountPercent,
			FinalPrice: finalPrice,
		}
	}
	return &dto.UserResponse{
		UserID:   user.UserID,
		Name:     user.Name,
		Email:    user.Email,
		Location: user.Location,
		Orders:   ordersResp,
	}, nil
}
```

### `internal/controller/user_controller.go`

```go
package controller

import (
	"database/sql"
	"errors"
	"go-clean-example/internal/usecase"
	"net/http"
	"strconv"
	"github.com/gin-gonic/gin"
)

type UserController struct {
	usecase usecase.UserUsecase
}

func NewUserController(usecase usecase.UserUsecase) *UserController {
	return &UserController{usecase: usecase}
}

func (h *UserController) GetUserWithOrders(c *gin.Context) {
	idParam := c.Param("id")
	userID, err := strconv.Atoi(idParam)
	if err != nil || userID <= 0 {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid ID"})
		return
	}

	limit, _ := strconv.Atoi(c.DefaultQuery("limit", "20"))
	offset, _ := strconv.Atoi(c.DefaultQuery("offset", "0"))
	if limit <= 0 {
		limit = 20
	}
	if offset < 0 {
		offset = 0
	}

	resp, err := h.usecase.GetUserWithOrders(c.Request.Context(), int32(userID), limit, offset)
	if errors.Is(err, sql.ErrNoRows) {
		c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
		return
	}
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to retrieve user with orders"})
		return
	}
	c.JSON(http.StatusOK, resp)
}
```

### `cmd/server/server.go`

```go
package main

import (
	"context"
	"database/sql"
	"go-clean-example/config"
	"go-clean-example/internal/controller"
	"go-clean-example/internal/repository/mysql"
	"go-clean-example/internal/usecase"
	"log"
	"net/http"
	"time"

	"github.com/adityaeka26/go-pkg/logger"
	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
	"go.uber.org/fx"
)

func newRouter(db *sql.DB) *gin.Engine {
	router := gin.New()
	router.Use(gin.Recovery())
	router.Use(cors.New(cors.Config{
		AllowOrigins:     []string{"*"},
		AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
		AllowHeaders:     []string{"Origin", "Content-Type", "X-User-ID"},
		ExposeHeaders:    []string{"Content-Length"},
		AllowCredentials: true,
	}))
	router.GET("/health/live", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"status": "alive"})
	})
	router.GET("/health/ready", func(c *gin.Context) {
		ctx, cancel := context.WithTimeout(c.Request.Context(), 2*time.Second)
		defer cancel()
		if err := db.PingContext(ctx); err != nil {
			c.JSON(http.StatusServiceUnavailable, gin.H{
				"status": "not ready",
				"error":  "database unreachable",
			})
			return
		}
		c.JSON(http.StatusOK, gin.H{"status": "ready"})
	})
	return router
}

func runServer(lc fx.Lifecycle, router *gin.Engine, db *sql.DB) {
	cfg := config.GetConfig()
	srv := &http.Server{
		Addr:    ":" + cfg.RestPort,
		Handler: router,
	}
	errCh := make(chan error, 1)
	lc.Append(fx.Hook{
		OnStart: func(ctx context.Context) error {
			go func() {
				if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
					errCh <- err
				}
			}()
			return nil
		},
		OnStop: func(ctx context.Context) error {
			// Check if server failed to start
			select {
			case err := <-errCh:
				log.Printf("Server failed to start: %v", err)
				return err
			default:
			}

			log.Println("Shutting down server...")
			shutdownErr := srv.Shutdown(ctx)
			if closeErr := db.Close(); closeErr != nil && shutdownErr == nil {
				return closeErr
			}
			if shutdownErr != nil {
				log.Printf("Server shutdown error: %v", shutdownErr)
			}
			log.Println("Database connection closed")
			return shutdownErr
		},
	})
}

func registerRoutes(router *gin.Engine, userController *controller.UserController) {
	user := router.Group("/api/users")
	{
		user.GET("/:id/orders", userController.GetUserWithOrders)
	}
}

func main() {
	app := fx.New(
		fx.Provide(
			logger.NewLogger,
			mysql.Open,
			mysql.NewUserRepository,
			usecase.NewUserUsecase,
			controller.NewUserController,
			newRouter,
		),
		fx.Invoke(registerRoutes, runServer),
	)
	app.Run()
}
```

## After creating all files

1. Run `go mod tidy`
2. Run `sqlc generate` — this produces `internal/repository/mysql/user/models.go`, `db.go`, `user.sql.go`
3. Build: `go build ./cmd/server/server.go`
4. Copy `.env.example` to `.env` and configure the DSN
5. Start MySQL, then run the app or `docker-compose up -d`
