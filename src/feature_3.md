# Feature 3
> Sebagai pengunjung saya ingin melihat daftar berita yang berisi judul, penulis, tanggal terbit dan isi berita

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