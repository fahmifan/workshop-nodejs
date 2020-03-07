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
    slug: String
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

### Content
Oke, sekarang kita akan membuat `content`, pertama kita tambahkan migration nya dulu.
Tambahkan `createTableContent` setelah `createTableUser` pada fungsi `migrate`. 
```js
let createTableContent = `
    CREATE TABLE IF NOT EXISTS contents (
        id INT NOT NULL AUTO_INCREMENT,
        author_id INT NOT NULL,
        title TEXT,
        slug VARCHAR(3072),
        body TEXT,
        CONSTRAINT contents_slug_unique UNIQUE (slug),
        PRIMARY KEY (id)
    )
`

utils.migrate([createTableUser, createTableContent])
```

Selanjutnya kita perlu membuat, repo nya. Tambahkan kode berikut
```js
// repository/content_repository.js
const mysql = require('../db/mysql')

exports.create = ({ author_id = 0, title = '', body = '' }) => {
    return new Promise((resolve, reject) => {
        pool.getConnection((err, conn) => {
            if (err) {
                console.error(err)
                reject(err)
                return
            }

            // slug ini harus dibuat unique
            const slug = createUniqueSlug(title)

            const q = `INSERT INTO contents (author_id, title, body, slug) VALUES(?, ?, ?, ?)`
            conn.query(q, [author_id, title, body, slug], (err, res) => {
                conn.release()

                if (err) {
                    console.error(err)
                    reject(err)
                    return
                }

                const id = res.insertedId
                resolve({ id, author_id, title, body, slug })
            })
        })
    })
}
```

Untuk membuat slug kita bisa menggunakan module `slugify`.
```
npm i slugify
```  

Lalu, kita buat fungsi `createUniqueSlug`, agar slug bisa selalu unique, kita akan tambahkan unix timestamp di akhir slug.
```js
// repository/content_repository.js

const slugify = require('slugify')
...
const createUniqueSlug = (text) => {
    const unique = (new Date()).getTime()
    return slugify(text, '-') + '-' + unique
}
...
```

Kita bisa coba dulu di file `test.js`
```js
const contentRepo = require('./repository/content_repository')
contentRepo.create({ author_id: 8, title: 'my first story', body: 'body of my first story'})
        .then(content => {
            console.log(content)
        })
        .catch(err => {
            console.error(err)
        })
        .finally(() => {
            process.exit(0)
        })
```

Sip, jika sudah berhasil, kita akan buat service nya.
```js
// service/content_service.js

const userRepo = require('../repository/user_repository.js')
const contentRepo = require('../repository/content_repository.js')

const Router = require('express').Router
const r = Router()

r.post("/", async (req, res) => {
    try {
        const { content } = req.body
        // TODO: add validation

        // check is user exists ?
        const user = await userRepo.findByID(content.author_id)
        if (!user || !user.id) {
            return res.status(404).json({"message": "author not found"})
        }

        const newContent = await contentRepo.create(content)
        if (!newContent) {
            return res.status(400).json({"message": "failed"})
        }
    
        return res.json(newContent)   
    } catch (err) {
        console.error(err)
        return res.status(500).json({ "message": "something wrong" })
    }
})

module.exports = r
```

Kita tambahkan `content_service` ke main router
```js
...
const contentService = require('./content_service')
...
app.use("/contents", contentService)
...
```

Wait, masih ada yang kurang, seharusnya hanya "user" yang boleh membuat content. Untuk itu kita perlu melakukan otentikasi client yang melakukan `create content`.

Kita perlu sebuah middleware untuk melakukan otentikasi. Untuk otentikasi kita akan menggunakan header `Authorization` dengan value `Bearer <YourToken>`. 
> client ---> middleware (bisa melakukan pencegatan untuk autentikasi) --> router
```js
// service/middleware.js

const auth = require('./auth.js')

exports.authenticate = (req, res, next) => {
    const bearerHeader = req.headers['authorization'];
    if (!bearerHeader) {
        // Forbidden
        return res.sendStatus(403);
    }

    const bearer = bearerHeader.split(' ');
    if (bearer.length != 2) {
        return res.status(403).json({"message": "invalid token"});
    }

    const bearerToken = bearer[1];
    const token = bearerToken;
    try {
        const decoded = auth.verfiy(token)
        if (!req.context) {
            req.context = {}
        }
    
        req.context.user = decoded  
        next();
    } catch (err) {
        console.error(err)
        return res.sendStatus(403)
    }
}

module.exports = exports
```

Lalu, kita pasang middleware tersebut ke content_service post
```js
const middleware = require('./middleware.js')
...
r.post("/", middleware.authenticate, async (req, res) => {...}
```

Next, kita akan melakukan update content. Untuk update content hanya dapat dilakukan oleh author nya saja dengan kata lain kita perlu cek autorisasi. Kita buat router nya dengan method PUT
```js
...
r.put("/",  middleware.authenticate, async(req, res) => {
    try {
        const { content } = req.body
        // TODO: add validation

        const { user } = req.context

        // check if content exists
        const oldContent = await contentRepo.findByID(content.id)
        if (!oldContent) {
            return res.status(404).json({"message": "content not found"})
        }

        // not the same user
        if (!user || (user.id !== oldContent.author_id)) {
            return res.status(403).json({"message": "cannot update content"})
        }

        const updatedContent = await contentRepo.update(content)
        if (!updatedContent) {
            return res.status(400).json({"message": "failed"})
        }
    
        return res.json(updatedContent)   
    } catch (err) {
        console.error(err)
        return res.status(500).json({"message": "something wrong"})
    }
})
...
```

Selanjutnya, kita perlu menampilkan daftar content yang ada. Kita buat repo nya.
Untuk mempermudah, pagination akan menggunakan `limit offset`.
```js
// repository/content_repository.js
...
const MAX_SIZE = 50
...
exports.findAll = ({ size = 0, page = 0 }) => {
    if (!page || page < 1) {
        page = 1
    }

    if (!size || size <= 0 || size > 50) {
        size = MAX_SIZE
    }

    // for first page, offset is 0
    const offset = size * (page-1)
    return new Promise((resolve, reject) => {
        pool.getConnection((err, conn) => {
            if (err) {
                console.error(err)
                reject(err)
                return
            }

            const q = `SELECT * FROM contents limit ? offset ?`
            conn.query(q, [size, offset], (err, res) => {
                if (err) {
                    console.error(err)
                    reject(err)
                    return
                }

                // transform res to array of simple object
                resolve(res.map(r => utils.objectifyRawPacket(r)))
            })
        })
    })
}
```

kita buat router nya di content_service
```js
r.get("/", async(req, res) => {
    try {
        let { size, page } = req.query

        // convert from string to number
        size = Number.parseInt(size, 10)
        page = Number.parseInt(page, 10)

        const contents = await contentRepo.findAll({ size, page })
        if (!contents) {
            return res.status(404).json({"message": "failed"})
        }
    
        return res.json(contents)   
    } catch (err) {
        console.error(err)
        return res.status(500).json({ "message": "something wrong" })
    }
})
```
