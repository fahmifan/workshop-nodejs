# Feature 2
> Sebagai pemilik platform berita hanya dapat ditulis/diubah oleh pengguna yg terotorisasi

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