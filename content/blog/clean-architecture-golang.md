---
title: "Implementasi Clean Architecture di Golang"
date: 2025-11-28
draft: false
tags: ["Go", "Architecture", "Backend", "Best Practices"]
categories: ["Tutorial"]
author: "wakwaw"
showToc: true
TocOpen: true
description: "Panduan lengkap menerapkan Clean Architecture di project Golang"
cover:
  image: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1200&h=630&fit=crop"
  alt: "Clean Architecture"
  caption: "Building maintainable Go applications"
  relative: false
---

## Introduction

Clean Architecture adalah konsep arsitektur software yang diperkenalkan oleh Robert C. Martin (Uncle Bob). Prinsip utamanya adalah **separation of concerns** dan **dependency inversion**.

## Why Clean Architecture?

- **Testability:** Setiap layer dapat di-test secara independen
- **Maintainability:** Perubahan di satu layer tidak mempengaruhi layer lain
- **Flexibility:** Mudah mengganti framework atau database

## Project Structure

```
myapp/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── domain/
│   │   └── user.go
│   ├── usecase/
│   │   └── user_usecase.go
│   ├── repository/
│   │   ├── user_repository.go
│   │   └── postgres/
│   │       └── user_postgres.go
│   └── delivery/
│       └── http/
│           └── user_handler.go
├── pkg/
│   └── utils/
├── go.mod
└── go.sum
```

## Layer by Layer

### 1. Domain Layer

Domain layer berisi **entities** dan **business rules**:

```go
// internal/domain/user.go
package domain

import "time"

type User struct {
    ID        int64     `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type UserRepository interface {
    Create(user *User) error
    GetByID(id int64) (*User, error)
    GetByEmail(email string) (*User, error)
    Update(user *User) error
    Delete(id int64) error
}

type UserUsecase interface {
    Register(email, name, password string) (*User, error)
    GetProfile(id int64) (*User, error)
}
```

### 2. Repository Layer

Repository mengimplementasikan interface untuk data persistence:

```go
// internal/repository/postgres/user_postgres.go
package postgres

import (
    "database/sql"
    "myapp/internal/domain"
)

type userRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) domain.UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(user *domain.User) error {
    query := `
        INSERT INTO users (email, name, created_at, updated_at)
        VALUES ($1, $2, NOW(), NOW())
        RETURNING id
    `
    return r.db.QueryRow(query, user.Email, user.Name).Scan(&user.ID)
}

func (r *userRepository) GetByID(id int64) (*domain.User, error) {
    user := &domain.User{}
    query := `SELECT id, email, name, created_at, updated_at FROM users WHERE id = $1`
    err := r.db.QueryRow(query, id).Scan(
        &user.ID, &user.Email, &user.Name, &user.CreatedAt, &user.UpdatedAt,
    )
    if err != nil {
        return nil, err
    }
    return user, nil
}
```

### 3. Usecase Layer

Usecase berisi **application business logic**:

```go
// internal/usecase/user_usecase.go
package usecase

import (
    "errors"
    "myapp/internal/domain"
)

type userUsecase struct {
    userRepo domain.UserRepository
}

func NewUserUsecase(ur domain.UserRepository) domain.UserUsecase {
    return &userUsecase{userRepo: ur}
}

func (u *userUsecase) Register(email, name, password string) (*domain.User, error) {
    // Check if email already exists
    existing, _ := u.userRepo.GetByEmail(email)
    if existing != nil {
        return nil, errors.New("email already registered")
    }

    user := &domain.User{
        Email: email,
        Name:  name,
    }

    if err := u.userRepo.Create(user); err != nil {
        return nil, err
    }

    return user, nil
}
```

### 4. Delivery Layer (Handler)

Handler menghandle HTTP requests:

```go
// internal/delivery/http/user_handler.go
package http

import (
    "encoding/json"
    "net/http"
    "myapp/internal/domain"
)

type UserHandler struct {
    usecase domain.UserUsecase
}

func NewUserHandler(uc domain.UserUsecase) *UserHandler {
    return &UserHandler{usecase: uc}
}

func (h *UserHandler) Register(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Email    string `json:"email"`
        Name     string `json:"name"`
        Password string `json:"password"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    user, err := h.usecase.Register(req.Email, req.Name, req.Password)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

## Dependency Injection

Semua dipasang di `main.go`:

```go
// cmd/api/main.go
package main

import (
    "database/sql"
    "log"
    "net/http"

    _ "github.com/lib/pq"
    
    "myapp/internal/delivery/http"
    "myapp/internal/repository/postgres"
    "myapp/internal/usecase"
)

func main() {
    // Database connection
    db, err := sql.Open("postgres", "postgres://...")
    if err != nil {
        log.Fatal(err)
    }

    // Initialize layers (dependency injection)
    userRepo := postgres.NewUserRepository(db)
    userUsecase := usecase.NewUserUsecase(userRepo)
    userHandler := http.NewUserHandler(userUsecase)

    // Routes
    mux := http.NewServeMux()
    mux.HandleFunc("POST /api/users/register", userHandler.Register)

    log.Println("Server running on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

## Testing Made Easy

Dengan Clean Architecture, testing jadi mudah menggunakan mock:

```go
// internal/usecase/user_usecase_test.go
package usecase_test

import (
    "testing"
    "myapp/internal/domain"
    "myapp/internal/usecase"
)

type mockUserRepo struct {
    users map[string]*domain.User
}

func (m *mockUserRepo) GetByEmail(email string) (*domain.User, error) {
    return m.users[email], nil
}

func (m *mockUserRepo) Create(user *domain.User) error {
    m.users[user.Email] = user
    user.ID = 1
    return nil
}

func TestRegister_Success(t *testing.T) {
    repo := &mockUserRepo{users: make(map[string]*domain.User)}
    uc := usecase.NewUserUsecase(repo)

    user, err := uc.Register("test@example.com", "Test", "password")
    
    if err != nil {
        t.Errorf("Expected no error, got %v", err)
    }
    if user.Email != "test@example.com" {
        t.Errorf("Expected email test@example.com, got %s", user.Email)
    }
}
```

## Conclusion

Clean Architecture memang butuh lebih banyak boilerplate, tapi manfaatnya sangat besar untuk project yang akan terus berkembang. Happy coding! 🚀

---

*Follow me di [@wakwaw](https://twitter.com/wakwaw) untuk tips software engineering lainnya.*
