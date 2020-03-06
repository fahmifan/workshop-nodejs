# Workshop Nodejs
## Start
Kita akan pakai `repository` pattern, jadi semua akses ke storage/thirdparty 
akan dilakukan di modul repository.
```
.
├── config.js
├── db
│   ├── migrations.js
│   ├── mysql.js
│   └── utils.js
├── main.js
├── package.json
├── package-lock.json
├── README.md
├── repository
│   └── user_repository.js
└── service
    ├── service.js
```

Buat folder project lalu buka terminal dan ketikkan
```
npm init -y
```

Lalu, kita install dependesi berikut
```
npm i express body-parser mysql nodemon
```

Buat service `service/service.js`
```js
const express = require('express')
const bodyParser = require('body-parser')

const app = express()

// create application/json parser
app.use(bodyParser.json({ type: 'application/json' }))
app.get("/ping", (req, res) => res.json({"ping": "pong"}))

exports.start = () => {
    app.listen(3000, () => {
        console.log("server start @ :3000")
    })
}

module.exports = exports
```

Kita buat file `main.js` di root folder
```js
const service = require('./service/service.js')

service.start()
```

Kita tes
```
curl localhost:3000/ping
```

## Features
1. Sebagai pengunjung saya ingin dapat mendaftarkan diri ke platform, menggunakan username dan password
2. Sebagai pemilik platform berita hanya dapat ditulis/diubah/dihapus oleh pengguna yg terotorisasi
3. Sebagai pengunjung saya ingin melihat daftar berita yang berisi judul, penulis, tanggal terbit dan isi berita


## Design
```
User {
    id: Int
    username: String
    password: String
    name: String
}

Content {
    id: Int
    author_id: Int
    title: String
    body: String
}
```

### Feature 1:
    - register --> check user exists
        - not exist --> create user
        - exists --> error 
    
    - login --> check user exists
        - not exists --> error
        - exists --> validate password 
            - invalid password --> error
            - valid --> logged in


## User service
Kita akan buat `user service` ini adalah salah satu core karena akan digunakan untuk
per-auth-an. Tapi sebelum nya kita akan setup database dulu. 
Buat file `db/mysql.js`
```js
const mysql = require('mysql');
let pool;
module.exports = {
    /**
     * @returns {mysql.Pool}
     */
    getPool: function () {
      if (pool) return pool;

      pool = mysql.createPool({
        host: '127.0.0.1',
        port: 3306,
        user: 'root',
        password: 'root',
        database: 'workshop'
      });

      return pool;
    }
};

```

Lalu kita buat `database migration`, kita bikin `utils` untuk jalanin migration.
Buat file `db/utils.js`
```js
// db/utils.js
const db = require('./mysql')
const pool = db.getPool()

exports.migrate = (queries = []) => {
    pool.getConnection((err, conn) => {
        if (err) {
            console.error(err)
            return
        }
    
        queries.forEach(q => {
            conn.query(q, (err, result) => {
                if (err) {
                    console.error(err)

                    conn.release()
                    break
                }
            })
        })

        conn.release()
        pool.end()
    })
}

module.exports = exports
```

Kita akan tulis migration2 nya di file `db/migrations.js`, kita perlu bikin user migration
```js
const utils = require('./utils.js')
let createTableUser = `
    CREATE TABLE IF NOT EXISTS users (
        id INT NOT NULL AUTO_INCREMENT,
        username VARCHAR(255),
        name VARCHAR(255),
        password TEXT,
        CONSTRAINT users_username_unique UNIQUE (username),
        PRIMARY KEY (id)
    );
`

utils.migrate([createTableUser])
```

sip, sebelum kita jalanin migrasi nya, nyalain mysql dan bikin database dengan nama `workshop`.
```
node db/migration.js
```

Sekarang kita buat `user_repository`, di sini lah fungsi CRUD dibuat. Untuk fitur satu, kita perlu 
fungsi `create user`
```js
const db = require('../db/mysql.js')
const pool = db.getPool()

exports.create = ({ username = '', name = '', password = '' }) => {
    return new Promise((resolve, reject) => {
        pool.getConnection((err, conn) => {
            if (err) {
                console.error(err)
                return
            }
            
            const q = `INSERT INTO users (username, name, password) VALUES(?, ?, ?)`
            conn.query(q, [username, name, password], (err, res) => {
                conn.release()

                if (err) {
                    console.error(err)
                    reject(err)
                    return
                }

                let id = res.insertId
                resolve({ username, name, id })
            })
        })
    })
}
```

Kita bisa test langsung buat file `test.js`
```js
const userRepo = require('./repository/user_repository.js')

userRepo.create({ username: "miun173", name: "fahmi", password: "test" })
    .then(user => {
        console.log(user)
    })
    .catch(err => {
        console.error(err)
    })
    .finally(() => {
        process.exit(0)
    })
```

Kalau udah berhasil terbuat dan tersimpan ke database. Kita lanjut bikin `user service`
```js
const userRepo = require('../repository/user_repository.js')

const Router = require('express').Router
const r = Router()

r.post("/", (req, res) => {
    const { user } = req.body
    // TODO: add validation
    const newUser = userRepo.create(user)
    if (!newUser) {
        res.status(400).json({"message": "failed"})
        return
    }
    res.json(newUser)
})

module.exports = r
```

Terus kita akan panggil `user_service.js` dari `service.js`
```js
// service/service.js
...
const userService = require('./user_service')
...
app.get("/ping", (req, res) => res.json({"ping": "pong"}))
app.use("/users", userService)
...
```

Sebelum kita tes, kita akan pakai nodemon untuk enable fitur hotreload. Kita akan buat script start.
Untuk jalanin `nodemon`
```json
// package.json
...
"scirpts": {
    ...
    "dev": "nodemon --ignore node_modules/ main.js"
}
...
```

selanjutnya tinggal jalanin
```
npm run dev
```

## Fitur 2
> Sebagai pemilik platform berita hanya dapat ditulis/diubah/dihapus oleh pengguna yg terotorisasi
Kita perlu mekanisme auth nih. Cara yg umum digunakan adalah token based auth. Dinama authentikasi dan authorisisai akan menggunakan token. Token ini sendiri bisa berupa string yang merepresentasikan data pengguna.

Kita akan buat token dengan jwt. Install dulu dependensi berikut
```
npm i jsonwebtoken
```

```js
// service/auth.js

var jwt = require('jsonwebtoken');

const secretKey = 'secret'

exports.sign = (user) => {
    if (user.password) {
        delete user.password
    }

    return jwt.sign(user, secretKey);
}

// return decoded token
exports.verfiy = (token) => {
    try {
        return jwt.verify(token, secretKey);
    } catch(err) {
        console.error(err)
        return false
    }
}

module.exports = exports
```


Untuk fitur login kita akan menggunakan username dan password dan mengembalikan jtw token
langsung saja kita buat repository untuk find by username nya. Krn hasil query mysql itu adalah sebuah class, kita perlu mentransform ke bentuk plain object.
Kita akan buat juga utility untuk transformasi nya
```js
// repository/utils.js
exports.objectifyRawPacket = row => ({...row});

module.exports = exports
```

```js
// repository/user_repository.js
...
exports.findByUsername = (username) => new Promise((resolve, reject) =>{
    pool.getConnection((err, conn) => {
        if (err) {
            reject(err)
            return
        }

        conn.query(`SELECT * FROM users WHERE username = ?`, [username], (err, res) => {
            conn.release()

            if (err) {
                console.error(err)
                reject(err)
                return
            }

            if (res.size == 0) {
                resolve(null)
                return
            }

            let obj = utils.objectifyRawPacket(res[0])
            console.log(obj)
            resolve(obj)
        })
    })
})
...
```

Lalu, kita buat auth_service nya
```js
//  service/auth_service.js
const userRepo = require('../repository/user_repository.js')
const auth = require('./auth.js')

const Router = require('express').Router
const r = Router()

r.post("/login", async (req, res) => {
    try {
        const { username, password } = req.body
        const user = await userRepo.findByUsername(username)
        if (!user || (user.password != password)) {
            return res.status(404).json({"message": "user not found"})
        }

        const token = auth.sign(user)
        return res.json(token)
    } catch (err) {
        console.error(err)
        res.status(403).json({"message": "unauthenticated"})
        return
    }
})

module.exports = r
```

Lalu, kita tambahkan ke service
```js
const authService = require('./auth_service') // import authService
...
app.use("/users", userService)
app.use("/auth", authService) // tambahkan di sini
...
```

Kita coba saja langsung di POSTMAN. Kalau sudah berhasil harusnya ada balikan token.
Token itu bisa dicek hasilnya dari verify.
```js
// test.js
const auth = require('./service/auth')
const decoded = auth.verfiy(`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6OCwidXNlcm5hbWUiOiJqb2huIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTgzNTA1NjQ1fQ.R7zMh8iLMau5q_yagjhFYYAtyqKLKP4B_KShh_rPjvQ`)

console.log(decoded)
```

