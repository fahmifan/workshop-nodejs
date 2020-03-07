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