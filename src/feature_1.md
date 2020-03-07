# Feature 1

> Sebagai pengunjung saya ingin dapat mendaftarkan diri ke platform, menggunakan username dan password

## Flow
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